
# 320B的GLM-5.3-Flash，我用四张R9700跑通了

在8月26号，智谱发布了GLM-5.3-Flash，它就是之前在OpenRouter上匿名测试的Ox Alpha、也是网友口中普遍称呼的牛来。GLM-5.3-Flash的效果非常不错，特别是在多模态方面。作为GLM系列的第一个原生多模态模型，它能把文本、图片、视频都作为输入处理，胜于前代。

多数爱好者现在都通过官方api来调用GLM-5.3-Flash，这次我尝试了在本地用llama.cpp跑它的开源版。不过，目前的推理框架的开源支持还在早期，想必大多数人会踩坑，因此分享一些我开源运行时模型和框架的选择经验：

1. 截至本文写作时，llama.cpp的main分支还不支持GLM-5.3-Flash，不过前沿GLM5Next分支支持。所以选择该分支
2. 量化版本，选择Unsloth刚好在GLM5Next分支下发布的 UD-Q2_K_XL。（UD-Q2_K_XL：简单理解，UD就是Unsloth Dynamic，是Unsloth出的动态量化方案，Q2_K笼统的说就是2-bit档位，XL是Unsloth 自己使用的量化档位后缀。）
3. 一个额外的的视觉模块权重文件mmproj-F16.gguf。

以下进行详细解释：

UD-Q2_K_XL能够把320B的原始权重压缩到一共108.7GB，然后再加一个1.13GB的mmproj-F16.gguf，就是最终的大小。四张 32GB 显存的卡理论上能装下这些。

这里额外说一下mmproj-F16.gguf。mmproj是multimodal projector的缩写，可以简单翻译为多模态投影模块。F16表示FP16半精度版本。

这是llama.cpp的多模态方案所额外需要的。在llama.cpp 的多模态方案中，通常需要两部分：

1. 主模型GGUF格式权重，负责语言理解、推理和文字生成。
2. 与模型配套的mmproj GGUF格式权重，负责把图片编码并投影成主模型能够接收的视觉嵌入，它对应的是链接/投影部分的权重。

因此，mmproj-F16.gguf一般被称为视觉投影层。

考虑模型的大小，本次我采用了四张Radeon AI PRO R9700，正好32GB × 4，通过llama.cpp的pipeline parallelism方式部署。这次主要测试了文字和图像，没有测试视频。因为截至本文写作时GLM5Next分支还没接通原生视频处理，只能暂时放弃。文字方面我测试了短回答的基础功能和长文本的检索性能。

具体的测试代码和步骤会在下文说明。

## 环境准备和前置检查

根据前文的计算，模型的权重和视觉模块加起来约 110GB，我们的四张 R9700 物理显存合计约 119.4GB。理论上有余量，但实际推理时KV cache和中间计算结果还要额外占空间，因此现实空间并不丰裕。

为什么用Vulkan而不是ROCm？因为在非Mi Instinct系列上，Vulkan一般比ROCm更加稳定。这次GLM5Next分支的Vulkan后端也可以直接驱动Radeon AI PRO R9700，非常方便。

下面是部署和测试过程：

环境准备：

- 四张Radeon AI PRO R9700，每张32GB
- RAM约502GiB
- 模型和视觉模块放在独立数据盘
- 推理后端Vulkan
- 以pipeline parallelism 方式加载模型
- Flash Attention 关闭（当前分支开启会降低这个模型的注意力精度）
- MTP 关闭

通过以下命令进行基础的空间检查：

```bash
df -h ./models
rocm-smi --showmeminfo vram --showpidgpus
```

需要至少留出120GB的空间来放模型。此外，rocm-smi 输出里，偶尔会看到使用 0 个 DRM 设备的瞬时 KFD 条目，不影响，看实际显存占用就行。

## 固定依赖版本

GLM5Next的分支和模型仓库都还在更新，不锁版本的话，同样的步骤隔几天结果可能不一样。因此我固定模型revision，和llama.cpp的commit。

模型在Unsloth发布的GLM-5.3-Flash-GGUF仓库下，将revision锁定在 2975ab414d30340466d8c51533c6e91f0cca64c1，然后是量化的UD-Q2_K_XL，和视觉模块的 mmproj-F16.gguf。

用以下命令下载。

```bash
    mkdir -p ./models/GLM-5.3-Flash-GGUF

    hf download unsloth/GLM-5.3-Flash-GGUF \
      --revision 2975ab414d30340466d8c51533c6e91f0cca64c1 \
      --include 'UD-Q2_K_XL/*' \
      --local-dir ./models/GLM-5.3-Flash-GGUF \
      --max-workers 2

    hf download unsloth/GLM-5.3-Flash-GGUF \
      mmproj-F16.gguf \
      --revision 2975ab414d30340466d8c51533c6e91f0cca64c1 \
      --local-dir ./models/GLM-5.3-Flash-GGUF
```

下载完成后查看目录，目录里应该有四个编号分片和一个 mmproj-F16.gguf。

接下来锁定commit。

```bash
git clone --single-branch \
    --branch glm5next/upstream \
    https://github.com/unslothai/llama.cpp.git

cd llama.cpp
git checkout --detach 949f7efb097eb20ef36fecdb1afaebff9a4ae7ed
git rev-parse HEAD
```

观察hash，结果和上面一致就对了。PS：文末附了PR链接。

## 构建Vulkan环境

构建在Ubuntu 24.04容器里完成，然后目录挂回宿主机

```bash
apt-get update
apt-get install -y \
    cmake build-essential ccache libvulkan-dev vulkan-tools \
    glslc glslang-tools libshaderc-dev spirv-tools spirv-headers

cmake -S . -B build-vulkan \
    -DGGML_VULKAN=ON \
    -DGGML_NATIVE=OFF \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLAMA_CURL=OFF

cmake --build build-vulkan \
    --config Release \
    -j32 \
    --target llama-cli llama-server llama-mtmd-cli
```

构建完成后，build-vulkan/bin下能看到llama-cli，llama-server和llama-mtmd-cli。

它们依赖同目录下的共享库，直接启动可能报找不到 libllama-cli-impl.so，因此再设动态库路径。

```bash
export LD_LIBRARY_PATH=$PWD/build-vulkan/bin${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
./build-vulkan/bin/llama-cli --list-devices
```

设备列表里能看到四张AMD Radeon AI PRO R9700，Vulkan0到Vulkan3。后面加载模型按这个顺序指定。

## 跑一个最小文本任务

先测试最短提示词的效果，来确认模型的基础功能打通。命令如下。

```bash
./build-vulkan/bin/llama-cli \
    -m ../models/GLM-5.3-Flash-GGUF/UD-Q2_K_XL/GLM-5.3-Flash-UD-Q2_K_XL-00001-of-00004.gguf \
    -dev Vulkan0,Vulkan1,Vulkan2,Vulkan3 \
    -sm layer \
    -ngl auto \
    -fit on \
    -fitt 2048 \
    -c 4096 \
    -t 32 \
    -tb 32 \
    -fa off \
    -n 512 \
    --temp 0 \
    --seed 0 \
    --reasoning-effort low \
    --reasoning-budget 256 \
    --single-turn \
    --no-mmproj \
    -p '请只用一句中文回答，本地 GLM-5.3-Flash 文本推理已经运行。'
```

参数说明：

- `-m` 模型文件路径，指向四个分片中的第一个，llama.cpp会自动找到剩余分片
- `-dev Vulkan0,Vulkan1,Vulkan2,Vulkan3` 指定四张GPU
- `-sm layer` pipeline parallelism对应的参数，把模型按层拆分到四张 GPU。
- `-ngl auto` 自动把所有层都放到 GPU，不留在 CPU
- `-fit on` 自动根据可用显存调整加载策略，防止爆显存
- `-fitt 2048` 每张GPU预留2048 MB安全余量
- `-c 4096` 上下文窗口大小
- `-t 32` / `-tb 32` 推理和批处理各用 32 个 CPU 线程
- `-fa off` 关闭Flash Attention，当前分支开启会降低这个模型的注意力精度
- `-n 512` 最多生成tokens数量
- `--temp 0` 温度为 0，每次输出固定结果
- `--seed 0` 固定随机种子，确保可复现
- `--reasoning-effort low` 思考强度设为低
- `--reasoning-budget 256` 决定推理用的tokens上限为256
- `--single-turn` 单轮对话，问完就退出
- `--no-mmproj` 不加载视觉模块，这一步只做纯文本

模型返回如下：

```text
本地GLM-5.3-Flash文本推理已成功运行。
```

下一步继续看文字能力，看模型能不能在更长的输入里准确找到信息。

## 验证长文本检索能力

之前的短回答只能证明模型加载成功、能生成文字。但是模型的文字理解效果需要专门测一下。这里我们测其中的一项上下文检索能力。

做法是构造一个检索任务。原理很简单：我们生成240条格式相同的记录，每条都是编号加一个随机码，肉眼看上去长得差不多。在第17条和第233条分别埋两个特殊验证码LANTERN-17017和 ORBIT-99233，然后把全部240条连同指令一起发给模型，要求它只返回这两个验证码。整段文本经过分词后大约7266个token，相当于几千字的中文文章，足够做注意力检测了。


命令和之前一样，只是prompt、输入输出长度不同，这里不再重复给出命令。

经过验证，模型返回结果一致

```text
    017=LANTERN-17017；233=ORBIT-99233
```

和预设完全一致，没有重复token或乱码。在低思考等级下表现不错。

## 加载视觉模块，测试静态图片

文本通过后，现在测试图片理解。

用下列图片进行测试，它是一张合成图片，顶部写着HELLO 42，左侧是蓝色正方形，右侧是红色圆形。

![本地视觉探针](images/vision-probe.png)

图片测试用llama-mtmd-cli而不是前面的 llama-cli，因为它支持加载视觉模块。

下列命令和前面的命令相比，主要多了 `-mm`（指定视觉模块路径）、`--image`（指定输入图片）和 `-mmdev`（视觉模块放在哪张卡上）三个参数，其他参数含义不变。

```bash
    ./build-vulkan/bin/llama-mtmd-cli \
      -m ../models/GLM-5.3-Flash-GGUF/UD-Q2_K_XL/GLM-5.3-Flash-UD-Q2_K_XL-00001-of-00004.gguf \
      -mm ../models/GLM-5.3-Flash-GGUF/mmproj-F16.gguf \
      --image ./vision-probe.png \
      -mmdev Vulkan0 \
      -dev Vulkan0,Vulkan1,Vulkan2,Vulkan3 \
      -sm layer \
      -ngl auto \
      -fit on \
      -fitt 2048 \
      -c 8192 \
      -t 32 \
      -tb 32 \
      -fa off \
      -n 512 \
      --temp 0 \
      --seed 0 \
      --jinja \
      -p '请读取图片，只用一句中文准确回答顶部文字、左侧形状及颜色、右侧形状及颜色。'
```

模型返回

```text
    顶部文字为"HELLO 42"，左侧是一个蓝色正方形，右侧是一个红色圆形。
```

结果正确。

## 测试真实视频帧

合成图片结构简单，所以我们再用一张真实画面尝试。之前我们8月18日在知乎发过一篇[《手把手教你在 AMD Radeon PRO W7900 上丝滑运行 MiniMax H3+ComfyUI 音视频生成》](https://zhuanlan.zhihu.com/p/2073092993117073920)，成片里是一台红黑机器人站在夜间实验室的工作站前。

我拿了那段视频的第36帧作为静态图片输入。

![MiniMax H3 视频第 36 帧](images/minimax-h3-frame-036.png)

命令和之前一样，只是变更相关参数大小，所以不列出。

结果，GLM-5.3-Flash识别出机器人有一只手臂伸向中央面板，判断中央圆形部件已经发出红光。这是正确的。但也有不对的：GLM-5.3-Flash同时写了手指已经触及中央控制面板，这只是模型的推断。


## 常见问题

### llama-mtmd-cli不认reasoning参数

llama-mtmd-cli不接受reasoning-effort、reasoning-budget、reasoning-format，加了会在模型加载前直接退出。只用llama-mtmd-cli --help里列出的参数就好。

### 帮助里有--video，能跑原生视频吗？

GLM5Next分支的源码里明确写了

```text
glm5next spells video with its own token pair, but video is not supported here
```

因此，模型专属的视频 token 还没接入。--video走的是通用媒体路径的拆帧，不是GLM-5.3-Flash的原生视频处理。

## 小结

本文用四张R9700以Vulkan后端按层拆分，加载了UD-Q2_K_XL量化的 GLM-5.3-Flash。文本端跑通了短回答和长内容检索，静态图片返回了正确内容。原生视频处理还没接入。

## 问你们的

有在其他Radeon上试过这个模型的吗？什么卡？用的哪个量化？跑了什么，效果如何？欢迎评论区聊聊。

## 参考资料

- [GLM-5.3-Flash 官方模型卡](https://huggingface.co/zai-org/GLM-5.3-Flash)
- [Unsloth GLM-5.3-Flash GGUF](https://huggingface.co/unsloth/GLM-5.3-Flash-GGUF)
- [llama.cpp GLM5Next 支持 PR](https://github.com/ggml-org/llama.cpp/pull/27754)
- [GLM5Next 视频限制源码](https://github.com/unslothai/llama.cpp/blob/949f7efb097eb20ef36fecdb1afaebff9a4ae7ed/tools/mtmd/mtmd.cpp#L863-L869)
