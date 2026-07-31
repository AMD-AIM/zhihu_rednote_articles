# A 卡跑 HuggingFace｜device 照样写 cuda

- **平台**：知乎
- **类型**：基础内容（第 1 篇）
- **来源**：ROCm / PyTorch / HuggingFace 官方文档整理 + 本机实测，按知乎风格改写
- **数据依据**：ROCm 7.2.0 本机实测（gfx1201 / RDNA4，2026-07-31）+ ROCm 7.14.0 官方安装页与 release notes（同日查证）。文中引用的 issue 编号、状态、官方原文均经独立核验，每条结论标注了版本与 open/已修状态
- **配图**：`images/` 下 5 张真机终端截图，已按 markdown 语法内嵌在正文对应位置

---

## 标题

**教你用A卡跑 HuggingFace 模型 [2026-07 实测·长期更新]**


## 正文

搜"A 卡 跑 HuggingFace"，你会发现几乎每篇教程的第一步都是同一句：

```bash
export HSA_OVERRIDE_GFX_VERSION=11.0.0
```

我一直以为这是装 ROCm 的标准动作。直到翻 AMD 自己的排障文档，看到人家写得很直白：**互联网上其他所有"ROCm 跑不起来"的教程都把这个变量当成第一个解法，我们故意把它放在最后。**

原因也讲得很清楚：如果你的卡本来就有原生 kernel，你却硬把它伪装成另一颗芯片，编译器会按伪装的目标生成指令，硬件执行的却是另一套指令集——运行时 page fault，或者干脆挂死。

这篇就是从头把这条路重走一遍的记录。**所有命令都是真机跑过的**，输出原样贴出来，版本号和日期都标清楚。毕竟，这个领域的教程半年就烂，你看到任何一篇不写版本号的，基本可以直接关掉。

本文实测环境：ROCm 7.2.0 / gfx1201（RDNA4）/ PyTorch 2.9.1 / transformers 5.14.1，2026 年 7 月 31 日。

---

### 一、先花十秒确认你的卡

在装任何东西之前，先搞清楚你手上这块卡的 gfx 号。后面装 PyTorch 要填它，出了问题查资料也要报它。

```bash
rocm-smi -d 0 --showproductname | grep -E "Card Series|Card Model|GFX Version"
```

`-d 0` 是只看 0 号卡，单卡机器加不加都一样；`grep` 是因为完整输出有十来行，真正要看的就下面三行：

```
GPU[0]  : Card Series:   AMD Radeon AI PRO R9700
GPU[0]  : Card Model:    0x7551
GPU[0]  : GFX Version:   gfx1201
```

![rocm-smi 输出：Card Series、Card Model 与 GFX Version](images/01-rocm-smi.png)

你多半还会先看到一行 `WARNING: AMD GPU device(s) is/are in a low-power state`。**这不是报错**，卡空闲的时候会掉进低功耗状态，`rocm-smi` 顺口提一句而已，不影响任何东西。它走的是 stderr，所以加了 `grep` 也照样冒出来——嫌碍眼就在管道前面加个 `2>/dev/null`。

或者更短的：

```bash
rocminfo | grep gfx
```

```
  Name:                    gfx1201
      Name:                    amdgcn-amd-amdhsa--gfx1201
      Name:                    amdgcn-amd-amdhsa--gfx12-generic
```

对照表，找你自己那一行：

| 架构 | gfx 号 | 代表型号 |
|---|---|---|
| RDNA4 | gfx1201 | RX 9070 XT / 9070 GRE / 9070、Radeon AI PRO R9700 |
| RDNA4 | gfx1200 | RX 9060 XT / 9060 |
| RDNA3 | gfx1100 | RX 7900 XTX / 7900 XT / 7900 GRE、PRO W7900 / W7800 |
| RDNA3 | gfx1101 | RX 7800 XT / 7700 XT、PRO W7700 |
| RDNA3 | gfx1102 | RX 7600 |
| RDNA3.5 | gfx1151 | Ryzen AI Max+ 395 / 392 / 388（Strix Halo） |
| RDNA2 | gfx1030 | PRO W6800 / V620 |

RX 6000 系列在这张表里只有 W6800 / V620 这种专业卡，游戏卡不在官方支持列表内，能不能跑得自己试。

顺便说一句系统。截至 ROCm 7.14.0（2026-07-15 发布），Radeon 这边官方验证过的是 Ubuntu 26.04 / 24.04.4 / 22.04.5、RHEL 10.2 / 9.8，以及 Windows 11 25H2。

这里有个坑值得单独提：**RHEL 10.1 和 9.7 是被这一版明确替换掉的**，release notes 原话是 "RHEL 10.2 replaces RHEL 10.1 as the validated RHEL 10 release"。你搜到的教程如果还写着 10.1 / 9.7，说明它至少落后一个版本了——顺带一提，我写这篇的初稿里也抄错成了 10.1，是回头核对官方 release notes 才发现的。这个领域就是这样，谁都不能凭记忆写。

另外 Ryzen AI Max 那条线跟独显不一样：只支持 Ubuntu 26.04 / 24.04.4 和 Windows，**没有 22.04.5，也没有 RHEL**。

你要是在 Arch、Fedora 或者某个 rolling release 上折腾，出问题的概率本身就高一截，这不是你操作的问题，是你选的战场问题。

---

### 二、装什么：现在有两条并行的发布流

AMD 现在同时维护着两条 ROCm 发布流：

| | 7.2.x 生产流 | 7.13 / 7.14 新流 |
|---|---|---|
| 文档位置 | radeon-ryzen 文档树，冻结在 7.2.1 | `rocm.docs.amd.com/en/latest/`，结构重组过 |
| Linux 装法 | 传统 `--index-url` | `repo.amd.com/rocm/whl-multi-arch/` + extras 语法 |
| Windows wheel | `repo.radeon.com/.../rocm-rel-7.2.1/` | — |

安装源、文档路径、包命名全都不一样。你搜到的教程如果是去年写的，用的一定是旧那套；官方新文档给的是新那套。两边混着用，装出来的东西认不出卡，很正常。

**新流的安装命令**长这样（以 gfx1100，也就是 7900 XTX 为例）：

```bash
python -m pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ \
  "torch[device-gfx1100]==2.12.0+rocm7.14.0" \
  "torchvision[device-gfx1100]==0.27.0+rocm7.14.0" \
  "torchaudio==2.11.0+rocm7.14.0"
```

方括号里换成你自己的 gfx 号就行：9070 XT 填 `device-gfx1201`，9060 XT 填 `device-gfx1200`，7800 XT 填 `device-gfx1101`，Ryzen AI Max+ 395 填 `device-gfx1151`。实在不确定，用 `torch[device-all]`，代价是包体积大一圈。

> 说明一下：**上面这条 7.14 的命令来自官方安装页（2026-07-31 查证），不是我实测的那个版本。** 

如果你不想碰宿主机环境，直接上官方 Docker 镜像更省事，本文所有实测就是在容器里做的：

```bash
docker run -it --device=/dev/kfd --device=/dev/dri \
  --group-add video --ipc=host --shm-size 8G \
  --security-opt seccomp=unconfined \
  rocm/pytorch:rocm7.2_ubuntu24.04_py3.12_pytorch_release_2.9.1
```

对应 7.14 的 tag 是 `rocm/pytorch:rocm7.14_ubuntu24.04_py3.12_pytorch_release_2.12.0`，命名规律一样，Ubuntu 版本和 Python 版本都能替换（注意 py3.14 只有 ubuntu26.04 的镜像）。

**一个提醒：别 pull `rocm/pytorch:latest`。** 我查的时候它最后更新停在 2026-05-28，还是 rocm7.2.4 时代的镜像，比带版本号的 tag 落后好几档。这个 tag 名字最像"最新"，实际最不是。**永远写全 tag。**

装完先验：

```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0))
print(torch.cuda.get_arch_list())
```

我这台的真实输出：

```
2.9.1+rocm7.2.0.git7e1940d4
True
AMD Radeon AI PRO R9700
['gfx908', 'gfx90a', 'gfx942', 'gfx1030', 'gfx1100', 'gfx1101',
 'gfx1200', 'gfx1201', 'gfx950', 'gfx1151', 'gfx1150']
```

![torch 版本、is_available、device name 与完整 arch list](images/02-arch-list.png)

**最后那行 `get_arch_list()` 是这篇文章里我最想让你记住的一条命令。** 它列的是你装的这个 wheel 里真正编译进去了哪些架构的 kernel。你的 gfx 号在这个列表里，就说明官方 wheel 原生支持你的卡——你不需要任何 override、任何伪装、任何环境变量。

看上面那串，gfx1100 / 1101 / 1151 / 1200 / 1201 全都在里面。也就是说从 7900 XTX 到 9070 XT 到 Strix Halo，**这一代消费级卡现在全都是原生支持的**。那些让你第一步先设 `HSA_OVERRIDE_GFX_VERSION` 的教程，写的时候可能是对的，现在已经过期了。

---

### 三、跑起来：device 就写 cuda

这是最反直觉的一点，很多人卡在这儿——总觉得 A 卡应该有个什么 `torch.rocm` 或者 `device='hip'`。

没有。HuggingFace 官方文档原话是：**AMD GPU 在 PyTorch 里上报的 device type 就是 `cuda`。** Transformers 在运行时检测 ROCm 并把算子路由到 AMD kernel，你不需要自己指定 device type。

所以你在 N 卡上抄来的代码，`device='cuda'`、`.cuda()`、`.to('cuda')`，一个字都不用改。

装两个包：

```bash
pip install transformers accelerate
```

跑一个最小的例子。我用的是 Qwen2.5-0.5B-Instruct，0.5B 权重小，下载快，验证够用：

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

真机跑完的结果：

```
cuda:0
显卡是计算机系统中用于处理和显示图像、视频和其他多媒体内容的专用硬件设备……
```

![pipeline 跑通，device 为 cuda:0，模型正常输出中文](images/03-pipeline-run.png)

下载加载一共 19.3 秒，生成 64 个 token 用了 1.84 秒，显存占用 1.00 GiB。就这么多。

**一个新版本的坑。** 上面那段代码我写的是 `dtype=torch.float16`，你在网上搜到的所有教程写的都是 `torch_dtype=`。transformers 5.x 改了参数名，老写法目前还能用，但会打一行提示：

```
`torch_dtype` is deprecated! Use `dtype` instead!
```

现在还只是警告，哪天真删了，你照着老教程抄就直接报错。但既然撞上了顺手记一笔。

---

### 四、模型下载不下来，这才是国内第一道坎

说实话，在国内跑 HuggingFace，ROCm 装不上都算是后面的问题。第一道坎是 `from_pretrained()` 拉不下权重。所有英文教程都默认你能直连，这一段对我们完全是空白。

常规做法是把 endpoint 换成国内镜像：

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HUB_DISABLE_XET=1
```

**但这里实际上有一个坑，而且报错信息会把你带到完全错误的方向上去，值得单独讲。**

我设了镜像之后跑 pipeline，报的是这个：

```
OSError: We couldn't connect to 'https://hf-mirror.com' to load the files,
and couldn't find them in the cached files.
Check your internet connection or see how to run the library in offline mode
```

按字面意思，这是网络不通。但我 curl 那个地址是 200，端口通、DNS 通、证书也没问题。折腾了一会儿把异常链完整打出来，底下真正的错误是这句：

```
huggingface_hub.errors.FileMetadataError: Distant resource does not seem to be
on huggingface.co. It is possible that a configuration issue prevents you from
downloading resources from https://huggingface.co. Please check your firewall
and proxy settings and make sure your SSL certificates are updated.
```

它让你查防火墙、查代理、查 SSL 证书。**这三样一样都不是。**

真实原因是：`huggingface_hub`（我这边是 1.26.0）下载文件之前会先发一个 HEAD 请求，检查响应头里有没有 `x-repo-commit`、`x-linked-etag` 这些 HF 特有的字段。拿它来确认"对面确实是一个 HF 服务器"。我对比了两边的响应头：

```
huggingface.co  →  HTTP/2 307
                   x-repo-commit: 7ae557604adf...
                   x-linked-etag: "0dbb161213629a23..."

hf-mirror.com   →  HTTP/2 308
                   location: https://huggingface.co/...
                   server: Caddy
                   （没有任何 HF 元数据字段）
```

![curl -sI 对比两边响应头，官方站有 x-repo-commit，镜像只回 308](images/04-header-diff.png)

镜像那边只回了一个 308 跳转，把请求原样转回官方站，自己不带元数据。`curl -L` 会乖乖跟着跳所以看起来是 200，但 `huggingface_hub` 拿到的响应里没有它要的字段，于是判定"这不是 HF"，直接报错。

这个现象和你的网络环境有关：如果你的机器本身能直连 huggingface.co，镜像站可能会直接把请求转发过去，反而触发上面的校验失败；对完全连不上 HF 的机器，镜像通常是正常工作的。

但下面这条判断是通用的，你可以直接拿去用：

> 只要你看到 `Distant resource does not seem to be on huggingface.co`，**别去查防火墙和证书**，去查你设的那个 endpoint 到底返回了什么头。用 `curl -sI` 看一眼有没有 `x-repo-commit`，没有就是镜像端的问题，换一个源。

几个可以换的方向：

- **ModelScope（魔搭）**：国内的模型站，很多热门模型都有同步，`pip install modelscope` 之后用它的 `snapshot_download` 拉到本地，再把本地路径喂给 `from_pretrained()`。从机制上说它整条下载链路都不经过 `huggingface_hub`，所以上面那个校验问题不会出现——不过这条我这次没实测，只是原理上应该绕得开。
- **`hf` 命令行先拉再离线加载**：先把整个 repo 下到本地目录，然后 `from_pretrained("/本地/路径")`。大权重推荐这么干，断了能续。
- **换别的镜像 endpoint**：换之前先 `curl -sI` 看一眼头，省得又踩上面那个坑。

---

### 五、踩坑手册

按我看到的撞击频率排的。

**1）认不出卡，先别急着设 override，去查包冲突。**

这个真的是重灾区。ROCm 仓库里有个 gfx1201 的案例，`rocminfo` 明明认得出卡，一初始化 HIP 就崩，报 `has 2 ISAs but can only support a single ISA`。查下来真因是 Ubuntu 发行版仓库里的 `libamdhip64-dev` 和 `libhsa-runtime64-1`（5.7.1 那个版本）在运行时把官方装的 7.2.1 顶掉了。把发行版自带的那几个包卸掉就好了，**一个 override 都不需要**。AMD 工程师在干净环境里是复现不出来的——这也说明它不是架构不支持，纯粹是环境里有两套 ROCm 在打架。

Arch 上也有个几乎一模一样的：7900 XT，ollama 死活 100% 跑 CPU，日志里 `"total vram"="0 B"`。用户设了 `HSA_OVERRIDE_GFX_VERSION=11.0.0` 毫无改善。最后发现是官方源和 AUR 的 `opencl-amd` 混装了，卸掉冲突的那个之后他自己的原话是 "Now it just works!! No need for any overrides"。

所以顺序应该反过来：**先查有没有两套 ROCm 打架，最后才考虑 override。**

**2）批量推理 + 左填充 + flash attention，在 RDNA3 上是雷区。**

这组合恰好是 HuggingFace 批量推理的标准写法，所以撞上的人不少。两条独立的 issue 落在同一片区域，但**它们的可解程度差很多，分开说**。

**第一条有明确的绕过办法。** 7900 XTX 上 `model.generate()` 做批量推理**永不返回**，GPU 满载但不报任何错。ROCm 6.3 是正常的，升到 7.2 之后回归。绕过就是关掉两个 SDPA 后端：

```python
torch.backends.cuda.enable_flash_sdp(False)
torch.backends.cuda.enable_mem_efficient_sdp(False)
```

触发条件比看上去窄：报告者说单独调 `F.scaled_dot_product_attention` 写最小复现反而**不触发**，必须是**左填充 + 带 KV cache 的自回归生成循环**才出得来。所以你要是照着最小例子测发现没事，别急着下结论。

**第二条到现在没有确定的解法，别指望网上那些说法。** transformers 官方的 PaddleOCR-VL 批量示例，配上 `flash_attention_2` + `padding_side='left'` + batch ≥ 8，在 7900 XTX 上直接把桌面和 GPU 一起搞崩。

我特意去翻了这条 issue 的完整讨论，因为想找出解法，结果是：报告者**从一开始就已经开了 Triton 后端的 flash attention**，后来换成 ROCm 官方的 flash-attention fork，batch 提到 64 **照样崩**；AMD 那边试了没能复现；讨论到后面转向了 GPU 频率和电源 profile，把 profile 设成 high 之后崩的频率降低了，但没有消除。

所以这条目前能说的只有"降 batch、避开这个组合"，**如果你看到哪篇文章说"改用 Triton 后端就好了"，那个说法在这条 issue 里是站不住的**。

**这两条到我写稿为止都还是 open 状态**，不是"已修复请升级"，心里有数。

**3）比崩溃更可怕的是不报错但算错。**

PyTorch 那边给这类问题专门有个标签叫 `module: correctness (silent)`——静默的正确性问题。你的程序跑完了，没有任何异常，结果是错的。图生成出来糊掉、模型胡言乱语，你还以为是模型不行。

已经出现过的几例：

- **gfx1201（RX 9070）上 `F.linear` 对某些输入算错**，出图直接糊。issue 在 2026-07-26 关闭，修复进了 ROCm 7.14，**坏掉的 kernel 是 `download.pytorch.org/whl/rocm7.2` 那个 wheel**。同样的代码在 7900 XTX 上不复现。
- **gfx1151 上 softmax 只返回一半**，`sum()` 得到 0.5 而不是 1.0。根因挺有意思：`C10_WARP_SIZE` 在 HIP 下被当成编译期常量 64，而 RDNA 实际的 warp size 是 32。这个已经修了。
- **gfx1151 上 head_dim=512 的 Flash SDPA 输出饱和到 512.0**，**仍 open**。这条要注意环境：报的是 **Windows 11 + AMD nightly 多架构 wheel**，而且只在 gfx1151 上出现，gfx1201 是正常的。

第一条我顺手在自己机器上验了一下，因为我这套环境正好是 ROCm 7.2。**20 组常见 shape（fp16 / fp32 各一遍）跟 CPU 对拍，相对误差最大 4.2e-04，全部正常，没复现。**

但这不代表 7.2 就没事，有个区别值得你知道：**我用的是 `rocm/pytorch` Docker 镜像里的 torch 构建，issue 指的是 `download.pytorch.org` 上的 pip wheel。两个都标着 7.2，但不是同一次构建。** 这大概能解释为什么这类问题总是有人遇到有人遇不到——报 bug 的时候把 `torch.__version__` 完整贴出来（我的是 `2.9.1+rocm7.2.0.git7e1940d4`，后面那串 git hash 才是真正区分构建的东西），比只说"我用的 7.2"有用得多。

所以每次换环境、换版本，花两秒钟跑个自检。上面两类问题各对应一行：

```python
import torch
import torch.nn.functional as F

# 1) softmax 数值正确性（对应 gfx1151 那个 warp size 的坑）
x = torch.randn(1, 1000, device="cuda")
print("softmax:", torch.softmax(x, dim=-1).sum().item())      # 应该是 1.0

# 2) F.linear 跟 CPU 对拍（对应上面第一条）
a = torch.randn(8, 4096, dtype=torch.float16)
w = torch.randn(4096, 4096, dtype=torch.float16)
ref = F.linear(a.float(), w.float())
got = F.linear(a.cuda(), w.cuda()).float().cpu()
print("linear 相对误差:", ((got - ref).abs().max() / ref.abs().max()).item())
```

我在 gfx1201 / ROCm 7.2.0 上的结果：softmax 是 `1.0`；linear 的相对误差在 `1e-4` 量级。

![自检输出：softmax 为 1.0，linear 相对误差 4e-4](images/05-softmax-check.png)

第二个数每次跑都不一样（输入是随机的），**看量级不看具体值**：fp16 在 1e-3 以内都算正常，fp32 应该到 1e-6。要是你跑出来是 0.1、1 这种量级，那就是真的算错了，别再往下调模型了，先去查版本。

**4）`expandable_segments` 这个开关，别当成通用优化无脑抄。**

`PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 在 N 卡教程里几乎是标配建议，抄到 A 卡上要小心：**gfx1201 上它会崩**，报 `hipErrorIllegalAddress`。对应的 PyTorch issue 在 2026-07-16 被 AMD 以 `not_planned` 关掉了——注意，**关闭不等于修好了**，只是不打算处理。

网上还流传另一个方向的说法，说 Strix Halo 用 GTT 大内存时这个开关是必需的、不加会分配失败。我去翻了源头：那句话出现在一条**讲显存上报的 issue 里、而且是在这条 issue 关闭半年之后由社区用户追加的评论**，前提还是配合他自己写的 `LD_PRELOAD` 补丁。**不是 AMD 的结论，也不是那条 issue 的主题。** 我手上没有 Strix Halo，没法验，所以只能告诉你这条的证据强度比看上去弱得多，别当定论。

Strix Halo 另外那个显存坑倒是实打实的：`hipMemGetInfo()` 上报的总显存只有 15.49 GiB，实际有 96 GiB，`torch.cuda.mem_get_info()` 跟着拿到错值，vLLM 据此算 KV cache 预算就会报假 OOM。**这个已经修了**——对应 issue 2026-05-29 关闭，修复 PR 是给 APU 的内存上报加上 GTT。旧版本上还会遇到的话，绕过办法是从 sysfs 读 `mem_info_gtt_total` 拿真实值。

**5）环境变量，该设的和千万别设的。**

| 变量 | 结论 | 说明 |
|---|---|---|
| `TORCH_BLAS_PREFER_HIPBLASLT=1` | 符合条件就设 | RX 7900 系列 / 7800 XT / Ryzen AI MAX 系列，在 PyTorch 2.14 之前跑**某些** LLM 推理负载会偏慢，官方点名的典型场景是 **vLLM FP16 decode 且 batch ≥ 8**。不是所有推理都受影响。设了会改走 hipBLASLt 后端，2.14 起变成这些架构的默认。这条藏在官方安装页最底下的 known issues 里 |
| `HSA_OVERRIDE_GFX_VERSION` | 默认别设 | 见开头。只有当你的卡确实没有原生 wheel、且存在一个足够接近的 gfx 时才用 |
| `PYTORCH_ROCM_ARCH` | 别设 | 这是编译期变量，对预编译的 wheel 在运行时完全无效。AMD 排障文档的原话是 "It is not a fix; it is misinformation." |
| `HIP_VISIBLE_DEVICES` | 按需 | 核显和独显共存时用来排掉核显。注意 PyTorch 的 HIP 构建同时也认 `CUDA_VISIBLE_DEVICES`，两个设成不同的值会出怪事 |
| `LD_LIBRARY_PATH` | 检查一下 | 如果指向系统的 `/opt/rocm/lib`，会和 wheel 自带的 HIP 打架。症状特别像版本不匹配，真因是加载顺序 |

**6）想跑量化的话，bitsandbytes 现在是能用的。**

维护者今年 7 月说明过，当前 PyPI 上的 wheel 已经 target 了 gfx1201，只要你的 PyTorch 是 ROCm 6.4 以上构建的，**不需要从源码编译**。

顺带纠正三条还在到处流传的假解法：

- `os.environ['BNB_BACKEND'] = 'rocm'` —— **完全无效**，bitsandbytes 根本不读这个变量。
- 手动改 PYTHONPATH 修 editable install —— 复现不出来。
- 用 `multi-backend-refactor` 分支 —— 已经废弃了，改用 main。

前两条是 AMD 的人在 gfx1201 / ROCm 7.2.3 上实测给的结论，第三条出自项目 contributor，没说在什么硬件上验的。分开讲是因为这三条的证据强度不一样，别一并当成"官方在同一台机器上逐条测过"。

还有个容易复发的 bug 提一句：当 ROCm 次版本号大于等于 10 的时候（比如 7.14），bitsandbytes 的版本解析会溢出，报 `libbitsandbytes_rocm84.so not found`。这个修过，但已经复发三次了，遇到先怀疑版本号。

---

### 六、什么时候你不该用 transformers

这篇讲的是 transformers 这条路，但它不是所有场景的最优解，说清楚边界比硬吹有用。

**如果你的目的就是本地跑个大模型聊天，那 GGUF 那条路（llama.cpp / Ollama / LM Studio）现在体验更好。** AMD 自己最新的消费级博客都是直接讲 GGUF 的，绕开了 transformers。原因不复杂：GGUF 生态的量化更成熟、显存占用更低、开箱即用，而 transformers 的价值在于你要改模型、要接训练、要用 HF 上那些还没人转成 GGUF 的新架构。

**另外两条路，消费级卡上我不建议碰：**

- **optimum-amd**：这个库目前半维护状态。提交历史从 2024 年 6 月直接跳到 2025 年 8 月，2026 年唯一一次提交是把 transformers 版本上限钉死，README 到现在还写着 "Install ROCm 5.7"。文档里说只支持 MI210 / 250 / 300。别把它当推荐方案。
- **TGI**：官方只支持 Instinct，消费级要跑得上 override hack，跟前面说的官方建议直接冲突。这条路对消费级还不成熟。

还有一个陷阱得点名：**`huggingface.co/amd` 那个页面上的安装教程停留在 ROCm 6.2，是 2024 年的东西。** 它是官方页面，所以特别有欺骗性。别照抄。

---

### 七、最后

现状是：官方文档里 HuggingFace 那一章白纸黑字写着"只在 Instinct 上验证过"，面向消费级的模型支持矩阵总共只有三个模型，其中还有一个链接是坏的。所以你在消费级 Radeon 上跑 transformers，本质上是走在一条官方没铺完的路上——这也是我写这篇的原因。

不过风向在变。HuggingFace 的工程师今年 7 月在一篇社区博客里写道，transformers 已经在 MI300 上跑每日 CI，**今年会把覆盖范围扩展到 Ryzen 和 Radeon**。这篇挂在个人命名空间下、标着 Community Article，不算官方公告，但作者本人就是往 transformers 里合入"把 AMD daily CI 切到 MI300"那个 PR 的人，可信度不低。真做成了，上面这些坑会少一大半。

这篇我会跟着 ROCm 版本长期更新。你踩到别的坑，或者上面哪条在你机器上结论不一样，评论区告诉我，我补进来的时候会写上是哪块卡哪个版本。

本文所有实测数据来自 ROCm 7.2.0 / gfx1201（RDNA4）/ PyTorch 2.9.1 / transformers 5.14.1，测试日期 2026-07-31，单卡环境。安装命令部分依据 ROCm 7.14 官方文档，同日查证。

#AMD #ROCm #HuggingFace #PyTorch #显卡 #深度学习 #大模型 #Radeon #人工智能

## 首评

知乎正文放外链影响流量，链接统一走评论区首评。以下为首评原文：

正文里提到的来源都在这儿，方便你自己核。文中每条结论我都尽量注明了状态（open / 已修 / 证据强度），链接是让你能翻到原始讨论，别只信我转述。

**官方文档**

- PyTorch on ROCm 安装页（gfx 号、安装命令、known issues 全以这里为准）：rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html
- ROCm 7.14.0 release notes（操作系统支持列表、RHEL 10.2 替换 10.1 的说明）：rocm.docs.amd.com/en/latest/about/release-notes.html
- HuggingFace 文档，"AMD GPU 的 device type 就是 cuda"那段：huggingface.co/docs/transformers/en/kernel_doc/loading_kernels
- AMD rocm-doctor 排障文档（HSA_OVERRIDE 为什么放最后、PYTORCH_ROCM_ARCH 那句 "it is misinformation" 都在这里，注意路径带 staging，属孵化中）：github.com/amd/skills → staging/rocm-doctor/reference.md
- 文末提到的那篇博客（HF 工程师发的 Community Article，不是官方博客栏目）：huggingface.co/blog/badaoui/transformers-on-amd-mi455


