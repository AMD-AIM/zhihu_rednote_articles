# 在 Radeon Cloud 上部署 MiniCPM-V：用 vLLM 起一个 OpenAI 兼容的多模态服务

<!--
内部交付元数据：
- **平台**：知乎
- **类型**：云端部署实操（第 4 篇）
- **来源**：AMD AI Developer Program / Radeon Cloud 页面操作与 vLLM OpenAI API 调用示例整理
- **范围**：使用 Radeon Cloud Template 部署 MiniCPM-V-4.6，并通过 OpenAI 兼容接口发起图片问答请求
-->

最近手头有几个任务要跑，本地又没有合适的卡，我一直在留意有没有能白嫖一点算力的地方。刷资料的时候刚好被推了一下 AMD 的 AI Developer Program，点进去看了看，就抱着"先试试"的心态注册了个账号，把想跑的东西丢上去，结果体验比我预期的顺。

ADP 给的其实是两类资源。一类是模型 API credits，如果你只是想调调 API，用里面的 Fireworks AI 那份就够了；另一类是 Radeon Cloud 的云端 GPU 环境，可以直接开一台带 AMD 卡的实例出来。我想搞清楚的是"模型在 AMD GPU 上到底怎么部署"，所以走的是后面这条路。

Radeon Cloud 用起来也比我想的简单：不用自己装 ROCm，不用编译 vLLM，把镜像和一条启动命令存成一个 Template，点 Launch，等它 ready，你就拿到了一个能直接发请求的 OpenAI 兼容服务。

这篇是我在一台 Radeon PRO W7900 实例上从头走一遍的记录：建 Template、部署 MiniCPM-V-4.6，最后发一张图片过去问一句话。命令都可以照抄。

先说结论，客户端这一侧我一行代码都没改。vLLM 在 ROCm 上把模型加载起来之后，暴露出来的就是标准的 `/v1/chat/completions`，`openai` SDK 和 `curl` 直接接上去就能用。

下面是我最后跑通的一套配置，你可以当模板抄：

| 配置项 | 值 |
|---|---|
| Container Image | `minicpm:vllm-rocm-w7900-minicpmv46-fix` |
| Deploy Type | `vLLM` |
| 模型 | `openbmb/MiniCPM-V-4.6` |
| 对外模型名 | `MiniCPM-V46` |
| 端口 | `8000` |
| 精度 | `bfloat16` |
| 请求地址 | `http://<服务地址>:8000/v1/chat/completions` |

我选 MiniCPM-V 纯粹是图它省事。它是 OpenBMB 开源的轻量级视觉语言模型（VLM），能同时吃文字和图片，体积小、加载快，很适合先把镜像、ROCm、vLLM、模型加载、OpenAI 接口这一整条链路整体验证一遍。等以后换更大的多模态模型，要改的也就是启动命令里的模型名和几个显存参数，流程本身是一样的。

![Radeon Cloud 页面入口](images/image-1.png)

### 一、我用的环境

- GPU：Radeon Cloud 上的 Radeon PRO W7900 实例，48GB 显存。
- 账号：已注册 AMD AI Developer Program（ADP），并且开通了容器和 GPU 权限。
- 推理框架：vLLM，平台镜像里已经带了，我没有自己装 ROCm 和 vLLM。
- 客户端：随便一台能联网的机器就行。

客户端上我只装了一个包：

```bash
python -m pip install openai
```

### 二、先进 Profile 页面

打开 Radeon Cloud 登录，进 Profile 页面。建 Template、启动实例、看 QuickStart，都在这个页面附近完成。

![Profile 页面](images/image-2.png)

这里提醒一句：容器和 GPU 权限要提前开通。没开通的话 Template 照样能保存，一切看起来都很正常，但后面点 Launch 会一直卡着，页面上还看不出是什么原因。

### 三、建 Template（这一步最关键）

Template 就是一份存起来的启动配置，镜像、部署方式、启动命令都在里面，下次重开实例不用再填一遍。保存 Template 本身不占 GPU，只有 Launch 才会真的按这份配置去创建实例，所以可以放心多存几个。

在 Profile 页面点 Add Template：

![添加 Template](images/image-3.png)

一共要填四项：

| 配置项 | 填写内容 |
|---|---|
| Template 名称 | 一个你自己认得出来的名字 |
| Container Image | `minicpm-v46-fix`，对应镜像 `minicpm:vllm-rocm-w7900-minicpmv46-fix` |
| Deploy Type | `vLLM` |
| Server Command | 见下面的启动命令 |

镜像里已经把跑 MiniCPM-V 要的基础环境准备好了，真正决定模型怎么起来的是 Server Command。我填的是这一串：

```bash
/usr/local/bin/vllm serve openbmb/MiniCPM-V-4.6 \
  --served-model-name MiniCPM-V46 \
  --trust-remote-code \
  --dtype bfloat16 \
  --max-model-len 262144 \
  --max-num-seqs 64 \
  --gpu-memory-utilization 0.9 \
  --host 0.0.0.0 \
  --port 8000
```

几个我觉得值得单独说一下的：

`vllm serve openbmb/MiniCPM-V-4.6` 让 vLLM 从 HuggingFace 把模型拉下来并启动 OpenAI 兼容服务。`--served-model-name MiniCPM-V46` 决定对外暴露的模型名，等会儿客户端请求里的 `model` 字段必须和它一模一样。`--trust-remote-code` 别漏，MiniCPM-V 的仓库里带了自定义模型代码，不加这个模型在加载阶段就直接失败。`--host 0.0.0.0 --port 8000` 让服务监听 8000 端口，QuickStart 后面给出的地址指的就是它。

另外四个参数都和显存有关，OOM 的时候就从这里往下调：

| 参数 | 作用 | 什么时候调整 |
|---|---|---|
| `--max-model-len 262144` | 最大上下文长度 | 显存不足时优先降低 |
| `--max-num-seqs 64` | 最大并发请求数 | 降低可以减少显存峰值 |
| `--gpu-memory-utilization 0.9` | vLLM 可使用的 GPU 显存比例 | OOM 时先降到 `0.8` |
| `--dtype bfloat16` | 推理精度 | W7900 保持 BF16 |

![填写 Template 配置](images/image-4.png)

填完点 Add Template 保存：

![保存 Template](images/image-5.png)

### 四、Launch，然后等

回到 Template 列表，在刚存的那个 Template 右侧点 Launch：

![启动 Template](images/image-6.png)

![确认启动实例](images/image-7.png)

这一步需要点耐心。多模态模型第一次拉起要走镜像准备、模型加载、服务初始化三个阶段，比后面几次重启慢得多。页面状态变成 ready 之前，你发请求过去只会拿到连接错误，别急着怀疑配置写错了。

### 五、从 QuickStart 里抄请求命令

实例 ready 之后打开 QuickStart。里面的请求命令已经把服务地址、端口和模型名都填好了，原样跑一遍就能确认服务到底正不正常。我一般都是先跑这个再写自己的代码。

![QuickStart 请求示例](images/image-8.png)

到这一步，这个实例就可以当成一台已经起好 vLLM 的远程 AMD GPU 机器看待了。后面的事情和你接任何一个 OpenAI 兼容服务没有任何区别。

### 六、真正发一次请求

服务地址长这样：

```text
http://<服务地址>:8000/v1/chat/completions
```

Python 里我用 `openai` 客户端：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://<服务地址>:8000/v1",
    api_key="EMPTY",
)

resp = client.chat.completions.create(
    model="MiniCPM-V46",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "图里有什么？"},
                {
                    "type": "image_url",
                    "image_url": {"url": "https://example.com/test.jpg"},
                },
            ],
        }
    ],
    stream=True,
)

for chunk in resp:
    print(chunk.choices[0].delta.content or "", end="")
```

`api_key="EMPTY"` 只是为了糊弄 OpenAI SDK 的参数校验，vLLM 默认根本不看这个值。`stream=True` 是我自己的习惯，让模型边生成边返回，在命令行里一眼就能分清是"服务已经开始吐字了"还是"卡在服务端了"。

如果你更喜欢 `curl`，等价写法是：

```bash
curl http://<服务地址>:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniCPM-V46",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "请描述这张图片的主要内容。"},
        {"type": "image_url", "image_url": {"url": "https://example.com/test.jpg"}}
      ]
    }]
  }'
```

请求体其实只有两块要注意：`model` 写 `MiniCPM-V46`，`messages` 里的 `content` 是个数组，文本和图片各占一项。图片可以传公网 URL，也可以照 QuickStart 的示例传 Base64。

返回里能看到和图片内容对得上的回答，这条链路就算通了：请求从我的机器发到 Radeon Cloud 实例，vLLM 接住 OpenAI 格式的消息，再把图片和文本交给 MiniCPM-V 去处理。

![请求返回结果](images/image-9.png)

### 七、发不通的时候，我一般按这个顺序查

带图的请求同时依赖实例状态、模型名和图片本身，一上来就闷头发很难判断是哪一层出的问题。我的习惯是分三步往下缩范围。

第一步，确认服务在、模型名没写错：

```bash
curl http://<服务地址>:8000/v1/models
# 返回的 data 里会包含 "id": "MiniCPM-V46"
```

第二步，发一次纯文本请求，把图片这条链路先摘出去：

```bash
curl http://<服务地址>:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"MiniCPM-V46","messages":[{"role":"user","content":"你好"}]}'
```

这两步都能过，那问题基本就锁死在图片这一层了：要么 URL 服务端拉不到，要么图片尺寸和上下文长度超了。这一步建议用小图配短问题去测，返回快，干扰项也少。

### 八、我踩过的几个坑

- **模型名大小写**。请求里的 `model` 要和 `--served-model-name` 完全一致，写成 `minicpm-v46` 会直接告诉你模型不存在。
- **第一次请求特别慢**。首次推理带 warmup 开销，比后面的请求慢一截，别拿它当稳定状态下的表现。
- **图片是服务端去拉的**。`image_url` 是由 vLLM 服务端下载的，不是客户端上传，所以内网地址和需要登录的链接都会失败，这类图只能走 Base64。
- **OOM 的调整顺序**。`--max-model-len` 影响最大，其次是 `--max-num-seqs`，最后才轮到 `--gpu-memory-utilization`。
- **`api_key` 不能留空**。OpenAI SDK 要求这个参数非空，vLLM 又不校验它的值，随手填个 `EMPTY` 就行。

### 九、用完记得清理

实例只要还在运行状态就会一直占着 GPU 配额，停止和删除都在实例管理页面操作。

Template 本身不占资源，留着下次直接 Launch 就好。想调上下文长度、并发数、显存比例这些参数的话，我建议另存成一个新的 Template，这样能和原来的配置横向对比。

这篇我只做到链路验证为止，吞吐和延迟都没测。等链路稳了，往上压 `--max-num-seqs` 看并发、拉高 `--max-model-len` 看长上下文，或者换量化权重降低显存压力，都能在同一个 Template 的基础上改。这部分的实测数据我后面会单独分享，欢迎关注 AMD算力极客社。

#AMD #RadeonCloud #ADP #ROCm #vLLM #MiniCPM #多模态 #大模型 #人工智能

参考资料：

- [AMD AI Developer Program](https://www.amd.com/en/developer/resources/ai-development.html)
- [vLLM OpenAI-Compatible Server](https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html)
- [vLLM Multimodal Inputs](https://docs.vllm.ai/en/latest/features/multimodal_inputs.html)
- [MiniCPM-V-4.6 模型页](https://huggingface.co/openbmb/MiniCPM-V-4_6)
