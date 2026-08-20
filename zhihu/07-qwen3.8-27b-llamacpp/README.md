# Qwen3.8 27B 开源了：一张大显存 A 卡如何用 llama.cpp 跑起来？

Qwen3.8 27B 开放权重之后，本地大模型又多了一个很有吸引力的选择。Qwen 官方把它定位为一个紧凑、适合部署的 270 亿参数 Dense 模型，同时强调编码、真实工作、长流程 Agent、图片与视频理解，以及可控制的思考模式。

AMD 也在模型发布时给出了 Day 0 支持：官方测试中的 Ryzen AI Max+ 395 和单张 Radeon AI PRO R9700 都已经能通过 llama.cpp 运行 Qwen3.8 27B。对 A 卡用户来说，接下来的问题就很直接了：自己的大显存显卡能不能跑，应该下载哪份模型，怎样把它起成真正可调用的本地服务？

我在 AMD Radeon Cloud 的一张 Radeon PRO W7900 上重新走了一遍完整链路。最后不仅让 llama.cpp 报告可 offload 的 65/65 层全部进入 GPU，也跑通了 `llama-cli`、`llama-server` 和 OpenAI 兼容 API。不开 MTP 时，几类不同输出的生成速度都稳定在 31.7 tok/s 左右；打开 MTP 后，自然文本到了 41.6 tok/s，代码和高可预测内容的持续生成测试则超过 60 tok/s。

![AMD Qwen3.8 27B Day 0 支持](images/amd-qwen38-day0-scorecard.jpg)

*AMD 官方 Day 0 preliminary 数据。图中的 up to 24 / 51 tok/s 是 24.5 / 51.8 的整数化展示，两组数据均来自 Windows、Vulkan 和至少三次运行的平均值，分别使用 MTP=4 与 MTP=2。硬件和配置不同，不能直接横向比较，也不是本文 W7900 的实测结果。*

### 先说清楚这张卡实际承担了什么

这次使用的是 Qwen3.8 27B 的 Q4_K_M GGUF，主模型文件约 19GB。为了测试模型自带的 Multi-Token Prediction，也就是 MTP，我还加载了一份约 1.68GB 的 Q4_0 draft。

不开 MTP 时，`llama-server` 实际占用约 17.82GiB 显存；打开 MTP=2 后是 19.16GiB，多出约 1.34GiB。高日志级别下，llama.cpp 明确报告可 offload 的 65/65 层都进入 W7900，其中包括输出层；另有约 682MiB 的 token embedding 等非层张量保持 CPU mapped。

本文只验证纯文本、8192 context 和单并发。Qwen3.8 27B 的原生 context 是 262K，也有原生视觉能力，但把模型跑起来不等于这些边界已经全部验证，所以这次不下载 projector，也不把 8192 的结果外推到 262K。

下面的兼容性结论只属于本文这张 W7900。官方给出的超过 24GB 显存门槛可以帮助判断模型是否装得下，不能据此宣布其他 AMD GPU 已经通过同一套 Linux + Vulkan 命令验证。

![W7900 实测环境](images/w7900-environment.png)

### 用固定版本的 llama.cpp 跑起来

这类新模型对运行时版本很敏感。本文固定在 llama.cpp commit `2e92ecd0247d25f09797f8fdb044a166522fc05d`，对应 build 10511。后续版本当然可能更快，但命令、模型和速度必须先绑定到同一个时间截面。

当前 Radeon Cloud 实例里直接 `git clone` 会遇到证书链问题。不要用 `-k` 或全局关闭 Git 的证书校验。GitHub 官方的 codeload 域名可以正常验证 TLS，因此这里直接下载固定 commit 的源码归档：

```bash
set -euo pipefail

mkdir -p ~/qwen38
cd ~/qwen38

curl -fL --retry 3 \
  "https://codeload.github.com/ggml-org/llama.cpp/tar.gz/2e92ecd0247d25f09797f8fdb044a166522fc05d" \
  -o llama.cpp-2e92ecd.tar.gz &&

echo "5767518d018f3a18ca8e7a6d1fcf928c61eecc665c002c919c4e6eefc43d8135  llama.cpp-2e92ecd.tar.gz" \
  | sha256sum -c - &&

tar -xzf llama.cpp-2e92ecd.tar.gz &&
cd llama.cpp-2e92ecd0247d25f09797f8fdb044a166522fc05d
```

补齐 Vulkan 构建依赖：

```bash
sudo apt update
sudo apt install -y \
  cmake build-essential ccache \
  libvulkan-dev vulkan-tools mesa-vulkan-drivers \
  glslc glslang-tools libshaderc-dev \
  spirv-tools spirv-headers
```

codeload 归档不带 Git 历史，默认构建会把 build identity 显示成 0 和 unknown。下面两项显式写回了本次实测的 build 和 commit，避免同一份代码在读者机器上丢失版本身份：

```bash
cmake -S . -B build-vulkan \
  -DGGML_VULKAN=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLAMA_BUILD_NUMBER=10511 \
  -DLLAMA_BUILD_COMMIT=2e92ecd0247d25f09797f8fdb044a166522fc05d \
  -DLLAMA_CURL=OFF \
  -DLLAMA_BUILD_UI=OFF \
  -DLLAMA_USE_PREBUILT_UI=OFF \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

cmake --build build-vulkan --config Release -j 32 \
  --target llama-cli llama-server llama-bench
```

这里关闭了内嵌 Web UI。原因不是 `llama-server` 不能用，而是 build 10511 默认还会去 Hugging Face 下载对应版本的 UI 资产，这条网络在 Radeon Cloud 上不可达。关闭 UI 后，命令行和 OpenAI 兼容 API 都不受影响。

编译完成后先锁定真正的离散 GPU：

```bash
export GGML_VK_VISIBLE_DEVICES=0

./build-vulkan/bin/llama-cli --version
./build-vulkan/bin/llama-cli --list-devices
```

当前环境同时能看到 W7900 和 llvmpipe。后者是 CPU 软件 Vulkan，若不锁设备，跑出一组数字也不代表模型真的用了显卡。正确输出中应当只有这一张目标卡：

```text
Vulkan0: AMD Radeon Graphics (RADV NAVI31) (49136 MiB, 49109 MiB free)
version: 0.1.2-dev (build 10511, commit 2e92ecd02)
```

### 下载 Q4 主模型

本文使用 ggml-org 从 Qwen 官方权重转换的 GGUF。Radeon Cloud 当前不能直连 Hugging Face，下面从可访问的镜像下载，再用 Hugging Face 官方 API 公布的 SHA-256 确认文件身份。下载 URL 也固定到本次核验的仓库 revision，而不是会继续移动的 `main`。

```bash
set -euo pipefail

cd ~/qwen38
mkdir -p models
cd models

curl -fL --retry 8 --retry-all-errors -C - \
  "https://hf-mirror.com/ggml-org/Qwen3.8-27B-GGUF/resolve/0669b98607d47046c7c2b3f801011d54a08cfccf/Qwen3.8-27B-Q4_K_M.gguf" \
  -o Qwen3.8-27B-Q4_K_M.gguf.part &&

echo "31629f53165ab6a7dad8c9847dcfd1fdf55829dac1e6e748f4a68581b0033d34  Qwen3.8-27B-Q4_K_M.gguf.part" \
  | sha256sum -c - &&

mv Qwen3.8-27B-Q4_K_M.gguf.part Qwen3.8-27B-Q4_K_M.gguf
```

先跑一条 CLI 命令，确认模型不是只完成了加载：

```bash
cd ~/qwen38/llama.cpp-2e92ecd0247d25f09797f8fdb044a166522fc05d

GGML_VK_VISIBLE_DEVICES=0 \
./build-vulkan/bin/llama-cli \
  -m ../models/Qwen3.8-27B-Q4_K_M.gguf \
  -ngl all \
  -c 4096 \
  -n 160 \
  --temp 0 \
  --jinja \
  --chat-template-kwargs '{"enable_thinking":false,"preserve_thinking":false}' \
  -cnv -st \
  -p "请只用一句中文确认：Qwen3.8 27B 已经在这张 AMD GPU 上成功运行。"
```

本次返回的是：

> Qwen3.8 27B 已经在这张 AMD GPU 上成功运行。

到这里，模型文件、chat template、Vulkan 后端和中文生成这四层才算一起通过。

### 再加入 MTP，把它起成一个本地 API

基础生成通过后，再下载 MTP draft：

```bash
set -euo pipefail
cd ~/qwen38/models

curl -fL --retry 8 --retry-all-errors -C - \
  "https://hf-mirror.com/ggml-org/Qwen3.8-27B-GGUF/resolve/0669b98607d47046c7c2b3f801011d54a08cfccf/mtp-Qwen3.8-27B-Q4_0.gguf" \
  -o mtp-Qwen3.8-27B-Q4_0.gguf.part &&

echo "051a1764cff8c4f3ee6ae8b00593a0364c7539c67fa50ffc58f3f96509fca38e  mtp-Qwen3.8-27B-Q4_0.gguf.part" \
  | sha256sum -c - &&

mv mtp-Qwen3.8-27B-Q4_0.gguf.part mtp-Qwen3.8-27B-Q4_0.gguf
```

MTP draft 会先提出少量候选 token，再由主模型验证；猜中的候选可以一次向前推进，猜不中则回到主模型结果。AMD 官方在独立显卡测试中使用 MTP=2，所以本文也从每轮最多 draft 2 个 token 开始。

终端里能回答一句话还不够。真正接入脚本、Agent 或应用，需要把模型起成服务：

```bash
cd ~/qwen38/llama.cpp-2e92ecd0247d25f09797f8fdb044a166522fc05d

GGML_VK_VISIBLE_DEVICES=0 \
./build-vulkan/bin/llama-server \
  -m ../models/Qwen3.8-27B-Q4_K_M.gguf \
  -md ../models/mtp-Qwen3.8-27B-Q4_0.gguf \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  -ngl all \
  -ngld all \
  --alias Qwen3.8-27B \
  -c 8192 \
  -np 1 \
  --jinja \
  --host 127.0.0.1 \
  --port 8080
```

如果只想先起一个不带 MTP 的 baseline API，删除 `-md`、`--spec-type`、`--spec-draft-n-max` 和 `-ngld all` 四项即可，其余模型、context、alias 与监听配置保持不变。

本文把服务绑定在 `127.0.0.1`。当前没有 API key，直接改成 `0.0.0.0` 会把一个无认证接口暴露出去，不应为了远程访问省掉安全边界。

另开一个 SSH 终端发请求：

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.8-27B",
    "messages": [{
      "role": "user",
      "content": "请用三句话回答：一个只有一张大显存 AMD GPU 的开发者，为什么会选择在本地运行 27B 大模型？不要使用标题或列表。"
    }],
    "temperature": 0,
    "top_p": 1,
    "seed": 20260819,
    "max_tokens": 256,
    "chat_template_kwargs": {
      "enable_thinking": false,
      "preserve_thinking": false
    }
  }'
```

这一请求连续运行 5 次，回复逐字一致，生成速度中位数是 44.7 tok/s。此时模型已经不只是命令行中的一次演示，而是一项可以被现有 OpenAI 客户端继续调用的本地能力。

### MTP 不是一个固定的翻倍开关

不开 MTP 时，四类输出的生成速度非常接近，都在 31.7–31.8 tok/s。打开 MTP=2 之后，差距才真正出现：

- 三句普通回答：44.7 tok/s，提升 41%；
- 自然说明：41.6 tok/s，提升 31%；
- 代码生成：64.0 tok/s，提升 102%；
- 连续整数：66.5 tok/s，提升 109%。

![W7900 上 MTP 的实测结果](images/mtp-speedup-w7900.png)

这四项都使用同一模型和 temperature=0；每一项内部的 baseline / MTP 对照使用同一个 prompt，各运行 5 次取中位数。代码和连续整数固定了输出上限，用来观察持续生成吞吐，不代表任务在上限内完整结束。

本次四项测试中，draft 接受率与加速幅度呈同向变化：连续整数的接受率是 100%，代码约 93.6%，而自然说明只有 43.5%。这说明输出可预测性会影响当前配置的 MTP 收益，但四个 prompt 还不足以把它写成所有任务的固定规律。

当前 Vulkan 路径下，三句回答和连续整数两边逐字一致，代码和自然说明则没有逐字一致。开启 MTP 后，仍然应该拿自己的任务检查输出，而不是默认获得 bit-exact parity。

显存代价倒是很集中。MTP=2 把服务占用从 17.82GiB 提高到 19.16GiB，增加约 1.34GiB。对 48GB 的 W7900 来说余量充足。

### 跑起来以后，再看它能不能完成工作

最后我让模型写一个 Python 函数：接收一组模型请求记录，忽略损坏数据，按模型统计请求数、平均延迟、P95 延迟和整体生成速度，并在代码末尾加入断言。

第一次把文件读取、健壮解析、命令行、统计和示例全部塞进同一个任务时，模型连续两次撞上了输出长度上限。把对象收缩到核心统计函数后，它用 1243 个 token 完成代码，耗时约 20 秒，生成速度 64.9 tok/s。去掉开头的 Markdown 围栏后，生成的 Python 在远端执行通过，正常输入、损坏记录、空输入和全无效输入四组断言全部成功。

它也留下了一个很具体的小瑕疵：开头写了 Markdown 代码围栏，结尾却漏掉了三个反引号。代码行为通过，格式纪律没有完全通过。比起只说模型「编码很强」，这个结果更接近实际开发中的状态：工作能完成，但输出仍然值得检查。

### 复现时，几个地方最容易卡住

- 源码：RC 上普通 GitHub clone 的证书链不正常，使用官方 codeload 固定 commit，不关闭证书校验。
- 模型：镜像解决下载入口，固定仓库 revision 与 SHA-256 才负责确认模型身份。
- 构建：build 10511 默认继续下载 Web UI 资产，RC 无法访问时应关闭 UI，只保留 API。
- 设备：Vulkan 同时列出 W7900 和 llvmpipe，必须用 `GGML_VK_VISIBLE_DEVICES=0` 锁定真实显卡。
- 存储：当前 Radeon Cloud Pod 没有持久化存储，日志和结果应及时保存到外部。

### 最后

这次从一台空的 Radeon Cloud 实例开始，Qwen3.8 27B 的 Q4_K_M 主模型、llama.cpp Vulkan 和 OpenAI 兼容 API 已经在单张 W7900 上完整连通。llama.cpp 报告可 offload 的 65/65 层全部进入 GPU；加入 MTP draft 后，服务总显存约 19.16GiB。

如果目标只是先把模型接进应用，CLI 和 localhost API 已经构成最短闭环；如果还希望降低等待时间，再加入 MTP，并用自己的任务测试收益。本次 900-token 截断代码吞吐约为 baseline 的 2.02 倍，连续整数截断测试约为 2.09 倍；自然说明的提升则约为 31%。

如果手上暂时没有合适的大显存显卡，也可以先从 [AMD AI Developer Program](https://www.amd.com/en/developer/resources/ai-development.html) 申请 AMD Radeon Cloud 资源，再按本文的 llama.cpp 路径运行。不同版本、量化和显卡上的结果可能很快变化；如果你跑出了不同数据，最有价值的是把模型、commit、后端和参数一起留下来。

#AMD #Qwen3.8 #llamacpp #Radeon #本地大模型 #人工智能

参考资料：

- [Qwen3.8 27B 官方模型页](https://huggingface.co/Qwen/Qwen3.8-27B)
- [ggml-org Qwen3.8 27B GGUF](https://huggingface.co/ggml-org/Qwen3.8-27B-GGUF)
- [AMD Qwen3.8 27B Day 0](https://www.amd.com/en/blogs/2026/run-qwen-3-8-27b-on-amd-ryzen-ai-max-and-radeon-graphics-cards-day-0.html)
- [llama.cpp MTP 支持 PR #22673](https://github.com/ggml-org/llama.cpp/pull/22673)
- [llama.cpp](https://github.com/ggml-org/llama.cpp)
