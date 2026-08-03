# A 卡跑 HuggingFace｜device 照样写 cuda

- **平台**：知乎
- **类型**：基础内容（第 1 篇）
- **来源**：ROCm / PyTorch / HuggingFace 官方文档整理 + 本机实测，按知乎风格改写
- **数据依据**：ROCm 7.2.0 本机实测（gfx1201 / RDNA4，2026-07-31）+ ROCm 7.14.0 官方安装页与 release notes（同日查证）
- **配图**：`images/` 下 5 张真机终端截图，已按 markdown 语法内嵌在正文对应位置

---

## 标题

**教你用A卡跑 HuggingFace 模型 [2026-07 实测·长期更新]**

## 正文

这篇从查卡、装环境、到跑通一个 HuggingFace 模型，全程按步骤来，末尾附常见坑的排查。实测环境 ROCm 7.2.0 / gfx1201（RDNA4）/ PyTorch 2.9.1 / transformers 5.14.1，2026 年 7 月 31 日。

先说一个很多人不知道的事。搜 A 卡跑 HuggingFace，几乎每篇教程第一步都是：

```bash
export HSA_OVERRIDE_GFX_VERSION=11.0.0
```

AMD 自己的排障文档对这个变量的态度是：所有教程都把它当第一个解法，我们故意把它放在最后。原因是，如果你的卡本来就有原生 kernel，硬把它伪装成另一颗芯片，编译器按伪装目标生成指令，硬件执行的却是另一套指令集，运行时会 page fault 或者挂死。

下面这篇文章会告诉你，现在的消费级卡基本都有原生 kernel 了，根本不需要设这个变量。

---

### 一、先确认你的卡

装任何东西之前，先查你的 gfx 号。后面装 PyTorch 要填它，出了问题查资料也要报它。

```bash
rocm-smi -d 0 --showproductname | grep -E "Card Series|Card Model|GFX Version"
```

![rocm-smi 查卡输出](images/01-rocm-smi.png)

或者更短：

```bash
rocminfo | grep gfx
```

对照表，找到你的卡对应的 gfx 号：

| 架构 | gfx 号 | 代表型号 |
|---|---|---|
| RDNA4 | gfx1201 | RX 9070 XT / 9070 GRE / 9070、Radeon AI PRO R9700 |
| RDNA4 | gfx1200 | RX 9060 XT / 9060 |
| RDNA3 | gfx1100 | RX 7900 XTX / 7900 XT / 7900 GRE、PRO W7900 / W7800 |
| RDNA3 | gfx1101 | RX 7800 XT / 7700 XT、PRO W7700 |
| RDNA3 | gfx1102 | RX 7600 |
| RDNA3.5 | gfx1151 | Ryzen AI Max+ 395 / 392 / 388（Strix Halo） |
| RDNA2 | gfx1030 | PRO W6800 / V620 |

RX 6000 系列在这张表里只有 W6800 / V620 两张专业卡，游戏卡不在官方支持列表内。

系统方面，截至 ROCm 7.14.0（2026-07-15 发布），Radeon 这边官方验证过的是 Ubuntu 26.04 / 24.04.4 / 22.04.5、RHEL 10.2 / 9.8，以及 Windows 11 25H2。Ryzen AI Max 那条线只支持 Ubuntu 26.04 / 24.04.4 和 Windows，没有 RHEL。

注意 RHEL 10.1 和 9.7 是被 ROCm 7.14.0 明确替换掉的旧版本。你搜到的教程如果还写着 10.1 / 9.7，说明它至少落后一个版本了。

---

### 二、装什么

AMD 现在同时维护两条 ROCm 发布流，装不上的第一大来源是两边的命令混着用：

| | 7.2.x 生产流 | 7.13 / 7.14 新流 |
|---|---|---|
| 文档位置 | radeon-ryzen 文档树，冻结在 7.2.1 | `rocm.docs.amd.com/en/latest/` |
| Linux 装法 | 传统 `--index-url` | `repo.amd.com/rocm/whl-multi-arch/` + extras 语法 |
| Windows wheel | `repo.radeon.com/.../rocm-rel-7.2.1/` | — |

新流的安装命令（以 gfx1100 / 7900 XTX 为例，来自官方安装页，2026-07-31 查证）：

```bash
python -m pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ \
  "torch[device-gfx1100]==2.12.0+rocm7.14.0" \
  "torchvision[device-gfx1100]==0.27.0+rocm7.14.0" \
  "torchaudio==2.11.0+rocm7.14.0"
```

方括号里换成你自己的 gfx 号：9070 XT 填 `device-gfx1201`，9060 XT 填 `device-gfx1200`，7800 XT 填 `device-gfx1101`，Ryzen AI Max+ 395 填 `device-gfx1151`。不确定架构的用 `torch[device-all]`，包体积会大一圈。

不想碰宿主机环境的话，官方 Docker 镜像更省事。本文所有实测都在容器里做的：

```bash
docker run -it --device=/dev/kfd --device=/dev/dri \
  --group-add video --ipc=host --shm-size 8G \
  --security-opt seccomp=unconfined \
  rocm/pytorch:rocm7.2_ubuntu24.04_py3.12_pytorch_release_2.9.1
```

对应 ROCm 7.14 的 tag 是 `rocm/pytorch:rocm7.14_ubuntu24.04_py3.12_pytorch_release_2.12.0`。注意 `rocm/pytorch:latest` 停在 2026-05-28，还是 rocm7.2.4 时代的镜像，tag 一定要写全。

装完先跑四行验证：

```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0))
print(torch.cuda.get_arch_list())
```

我这台的输出：

```
2.9.1+rocm7.2.0.git7e1940d4
True
AMD Radeon AI PRO R9700
['gfx908', 'gfx90a', 'gfx942', 'gfx1030', 'gfx1100', 'gfx1101',
 'gfx1200', 'gfx1201', 'gfx950', 'gfx1151', 'gfx1150']
```

![验证输出与完整 arch list](images/02-arch-list.png)

`get_arch_list()` 列出的是这个 wheel 里编译进去了哪些架构的 kernel。你的 gfx 号在这个列表里，就说明官方 wheel 原生支持你的卡，不需要设 `HSA_OVERRIDE_GFX_VERSION`。

gfx1100 / 1101 / 1151 / 1200 / 1201 全在里面。从 7900 XTX 到 9070 XT 到 Strix Halo，这一代消费级卡都是原生支持的。让你第一步先设 override 的教程，写的时候可能是对的，现在已经过期了。

---

### 三、跑起来

A 卡在 PyTorch 里的 device type 就是 `cuda`。HuggingFace 官方文档原话：AMD GPU 在 PyTorch 里上报的 device type 就是 cuda，Transformers 在运行时检测 ROCm 并把算子路由到 AMD kernel。

你在 N 卡上写的代码，`device='cuda'`、`.cuda()`、`.to('cuda')`，直接搬过来就能跑。

装两个包：

```bash
pip install transformers accelerate
```

跑一个最小例子，用 Qwen2.5-0.5B-Instruct，0.5B 权重小，下载快，验证够用：

```python
import torch
from transformers import pipeline

pipe = pipeline(
    "text-generation",
    model="Qwen/Qwen2.5-0.5B-Instruct",
    dtype=torch.float16,
    device="cuda",
)

print(pipe.model.device)

out = pipe(
    [{"role": "user", "content": "用一句话说明什么是显卡。"}],
    max_new_tokens=64,
    do_sample=False,
)
print(out[0]["generated_text"][-1]["content"])
```

输出：

```
cuda:0
显卡是计算机系统中用于处理和显示图像、视频和其他多媒体内容的专用硬件设备……
```

![pipeline 跑通，device 为 cuda:0](images/03-pipeline-run.png)

下载加载一共 19.3 秒，生成 64 个 token 用了 1.84 秒，显存占用 1.00 GiB。

注意上面代码里写的是 `dtype=torch.float16`，网上教程写的都是 `torch_dtype=`。transformers 5.x 改了参数名，老写法目前还能用但会打 deprecation 警告，后续版本可能会删掉。

---

### 四、模型下载不下来

在国内跑 HuggingFace，ROCm 装不上都算是后面的问题。第一道坎是 `from_pretrained()` 拉不下权重。

常规做法是把 endpoint 换成国内镜像：

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HUB_DISABLE_XET=1
```

我设了镜像之后跑 pipeline，报了这个错：

```
OSError: We couldn't connect to 'https://hf-mirror.com' to load the files,
and couldn't find them in the cached files.
```

curl 那个地址是 200，端口通、DNS 通、证书也没问题。把异常链完整打出来，底层的错误是：

```
huggingface_hub.errors.FileMetadataError: Distant resource does not seem to be
on huggingface.co.
```

`huggingface_hub`（1.26.0）下载前会先发一个 HEAD 请求，检查响应头里有没有 `x-repo-commit`、`x-linked-etag` 这些字段，拿来确认对端是 HF 服务器。两边的响应头对比：

```
huggingface.co  →  HTTP/2 307
                   x-repo-commit: 7ae557604adf...
                   x-linked-etag: "0dbb161213629a23..."

hf-mirror.com   →  HTTP/2 308
                   location: https://huggingface.co/...
                   server: Caddy
```

![两边响应头对比](images/04-header-diff.png)

镜像只回了一个 308 跳转，不带元数据。`curl -L` 会跟着跳所以看起来是 200，但 `huggingface_hub` 拿到的响应里没有它要的字段，直接拒绝。

这个和网络环境有关：如果你的机器本身能直连 huggingface.co，镜像站可能会直接把请求转发过去，反而触发校验失败。对完全连不上 HF 的机器，镜像通常正常工作。

通用的排查方式：看到 `Distant resource does not seem to be on huggingface.co`，用 `curl -sI` 看你设的那个 endpoint 有没有返回 `x-repo-commit`，没有就换源。

几个替代方案：

- ModelScope（魔搭）：`pip install modelscope` 之后用 `snapshot_download` 拉到本地，再把本地路径传给 `from_pretrained()`
- `hf` 命令行先拉再离线加载：先把整个 repo 下到本地目录，然后 `from_pretrained("/本地/路径")`
- 换别的镜像 endpoint：换之前先 `curl -sI` 查一下头

---

### 五、踩坑手册

按撞击频率排。

1）认不出卡，先查包冲突，不要直接设 override。

ROCm 仓库里有个 gfx1201 的案例：`rocminfo` 认得出卡，一初始化 HIP 就崩，报 `has 2 ISAs but can only support a single ISA`。真因是 Ubuntu 发行版仓库里的 `libamdhip64-dev` 和 `libhsa-runtime64-1`（5.7.1）在运行时把官方装的 7.2.1 顶掉了。卸掉发行版自带的那几个包就好了。AMD 工程师在干净环境里复现不出来，说明不是架构不支持，是环境里有两套 ROCm 在打架。

Arch 上也有类似的：7900 XT，ollama 100% 跑 CPU，日志里 total vram 报 0。用户设了 `HSA_OVERRIDE_GFX_VERSION=11.0.0` 没有改善。最后发现是官方源和 AUR 的 `opencl-amd` 混装了。卸掉冲突方后问题消失。

2）批量推理 + 左填充 + flash attention 在 RDNA3 上是雷区。

这组合是 HuggingFace 批量推理的标准写法。两条独立的 issue 落在同一片区域，可解程度差很多。

第一条：7900 XTX 上 `model.generate()` 做批量推理永不返回，GPU 满载但不报错。ROCm 6.3 正常，升到 7.2 后回归。绕过方式是关掉两个 SDPA 后端：

```python
torch.backends.cuda.enable_flash_sdp(False)
torch.backends.cuda.enable_mem_efficient_sdp(False)
```

触发条件比较窄：单独调 `F.scaled_dot_product_attention` 写最小复现不触发，必须是左填充 + 带 KV cache 的自回归生成循环。

第二条：transformers 官方的 PaddleOCR-VL 批量示例，配上 `flash_attention_2` + `padding_side='left'` + batch ≥ 8，在 7900 XTX 上直接把桌面和 GPU 一起搞崩。issue 里报告者从一开始就开了 Triton 后端，后来换成 ROCm 官方的 flash-attention fork，batch 提到 64 照样崩。AMD 侧未能复现。目前只能降 batch 或者避开这个组合。

这两条到 2026-07-31 都是 open 状态。

3）不报错但算错。

PyTorch 管这类问题叫 `module: correctness (silent)`。程序正常跑完，结果是错的。

已经出现过的几例：

- gfx1201（RX 9070）上 `F.linear` 对某些输入算错，出图直接糊。2026-07-26 关闭，修复进了 ROCm 7.14，出问题的 kernel 在 `download.pytorch.org/whl/rocm7.2` 那个 wheel 里。
- gfx1151 上 softmax 只返回一半，`sum()` 得到 0.5。根因是 `C10_WARP_SIZE` 在 HIP 下取编译期常量 64，而 RDNA 实际 warp size 是 32。已修。
- gfx1151 上 head_dim=512 的 Flash SDPA 输出饱和到 512.0。仍 open。注意这条是 Windows 11 + AMD nightly 多架构 wheel 环境，只在 gfx1151 上出现。

第一条我在自己机器上验了一下（ROCm 7.2），20 组常见 shape（fp16 / fp32 各一遍）跟 CPU 对拍，相对误差最大 4.2e-04，没复现。不过我用的是 `rocm/pytorch` Docker 镜像里的 torch 构建，issue 指的是 `download.pytorch.org` 上的 pip wheel，两个都标着 7.2 但不是同一次构建。报 bug 的时候贴完整的 `torch.__version__`（比如 `2.9.1+rocm7.2.0.git7e1940d4`），后面那串 git hash 才是真正区分构建的东西。

换环境的时候跑个自检：

```python
import torch
import torch.nn.functional as F

x = torch.randn(1, 1000, device="cuda")
print("softmax:", torch.softmax(x, dim=-1).sum().item())

a = torch.randn(8, 4096, dtype=torch.float16)
w = torch.randn(4096, 4096, dtype=torch.float16)
ref = F.linear(a.float(), w.float())
got = F.linear(a.cuda(), w.cuda()).float().cpu()
print("linear 相对误差:", ((got - ref).abs().max() / ref.abs().max()).item())
```

softmax 应该是 1.0。linear 的相对误差每次跑都不一样（输入是随机的），fp16 在 1e-3 以内正常，fp32 应该到 1e-6。如果你跑出来是 0.1 这种量级，先查版本。

![自检输出](images/05-softmax-check.png)

4）`expandable_segments` 别从 N 卡教程照抄。

`PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 在 N 卡教程里是标配建议，gfx1201 上会崩，报 `hipErrorIllegalAddress`。对应 issue 2026-07-16 被 AMD 以 not_planned 关掉，不是修好了，是不打算处理。

5）环境变量速查。

| 变量 | 结论 | 说明 |
|---|---|---|
| `TORCH_BLAS_PREFER_HIPBLASLT=1` | 符合条件就设 | RX 7900 系列 / 7800 XT / Ryzen AI MAX 系列，PyTorch 2.14 之前跑 vLLM FP16 decode 且 batch ≥ 8 时偏慢，设了改走 hipBLASLt 后端。2.14 起成为这些架构的默认 |
| `HSA_OVERRIDE_GFX_VERSION` | 默认别设 | 见开头 |
| `PYTORCH_ROCM_ARCH` | 别设 | 编译期变量，对预编译 wheel 运行时无效。AMD 排障文档原话：It is not a fix, it is misinformation. |
| `HIP_VISIBLE_DEVICES` | 按需 | 核显和独显共存时排掉核显 |
| `LD_LIBRARY_PATH` | 检查一下 | 指向 `/opt/rocm/lib` 会和 wheel 自带的 HIP 冲突 |

6）bitsandbytes 现在能用。

维护者今年 7 月说明过，当前 PyPI 上的 wheel 已经 target 了 gfx1201，PyTorch 是 ROCm 6.4 以上构建的就不需要从源码编译。

三条流传的假解法：`os.environ['BNB_BACKEND'] = 'rocm'` 无效，bitsandbytes 不读这个变量；`multi-backend-refactor` 分支已废弃，用 main；手动改 PYTHONPATH 修 editable install 复现不出来。

ROCm 次版本号 ≥ 10 时（比如 7.14）bitsandbytes 版本解析会溢出，报 `libbitsandbytes_rocm84.so not found`。修过但复发过三次，遇到先怀疑版本号。

---

### 六、什么时候不该用 transformers

如果目的是本地跑大模型聊天，GGUF 那条路（llama.cpp / Ollama / LM Studio）体验更好。AMD 自己最新的消费级博客都是讲 GGUF 的。GGUF 量化更成熟、显存占用更低、开箱即用。transformers 的价值在于你要改模型、接训练、或者用 HF 上那些还没有 GGUF 版的新架构。

消费级卡上不建议碰的两条路：

- optimum-amd：半维护状态，提交历史从 2024 年 6 月跳到 2025 年 8 月，2026 年唯一一次提交是把 transformers 版本上限钉死，README 还写着 Install ROCm 5.7。
- TGI：官方只支持 Instinct，消费级要跑得靠 override hack。

`huggingface.co/amd` 页面上的安装教程停留在 ROCm 6.2，是 2024 年的东西，别照抄。

---

### 七、最后

官方文档里 HuggingFace 那一章写着只在 Instinct 上验证过，面向消费级的模型支持矩阵总共三个模型。在消费级 Radeon 上跑 transformers，目前走的是一条官方没铺完的路。

HuggingFace 的工程师今年 7 月写过，transformers 已经在 MI300 上跑每日 CI，今年会把覆盖范围扩展到 Ryzen 和 Radeon。

这篇会跟着 ROCm 版本长期更新。踩到别的坑，或者上面哪条在你机器上结论不一样，评论区告诉我。

本文实测数据来自 ROCm 7.2.0 / gfx1201（RDNA4）/ PyTorch 2.9.1 / transformers 5.14.1，2026-07-31，单卡。安装命令部分依据 ROCm 7.14 官方文档，同日查证。

#AMD #ROCm #HuggingFace #PyTorch #显卡 #深度学习 #大模型 #Radeon #人工智能

## 首评

正文里提到的来源：

- PyTorch on ROCm 安装页：rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html
- ROCm 7.14.0 release notes：rocm.docs.amd.com/en/latest/about/release-notes.html
- HuggingFace 文档：huggingface.co/docs/transformers/en/kernel_doc/loading_kernels
- AMD rocm-doctor 排障文档：github.com/amd/skills → staging/rocm-doctor/reference.md

什么卡、卡在哪一步，评论区说。
