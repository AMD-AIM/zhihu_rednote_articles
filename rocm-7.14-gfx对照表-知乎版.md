# AMD GPU / APU 架构对照表（ROCm 7.14.0）-- 判断ROCm 7.14.0 该装哪个包

先说结论，急的人看完这四条就可以走了：

1. **装包时你要找的只有一个值——你的 GPU 对应的 LLVM target（也就是 `gfx` 号）。** 它就是安装命令里 `[device-gfxXXXX]` 那个 extras 标签。
2. **7.14.0 起没有分架构的 index URL 了。** 过去要记 `whl/gfx94X-dcgpu/`、`whl/gfx110X-all/`、`whl/gfx1151/` 一堆路径，现在统一成 `https://repo.amd.com/rocm/whl-multi-arch/` 一个索引，架构靠 extras 选。
3. **`pip install torch` 不带 extras 是空装。** 装出来的是纯 host 代码、零 device 代码的 torch，能 `import`，`torch.cuda.is_available()` 给你 False，然后你会花一晚上怀疑是驱动问题。这是新机制下最容易踩的坑，下面第二节细说。
4. **gfx 号只决定装什么包，不决定你的系统能不能跑。** 尤其是 WSL——同一个 gfx 号里，不同型号的结论都不一样。第三节细说。

下面按「查 gfx 号 → 拼安装命令 → 核对 OS」三步走，每步一张表。

---

## 一、第一步：查到你的 gfx 号

四张表按产品线分。找到你的型号，记住 `LLVM Target` 那一列。

### Instinct 系列（数据中心）

| 设备系列 | 具体型号 | LLVM Target | 架构 |
| --- | --- | --- | --- |
| MI350 Series | MI355X, MI350X, MI350P | `gfx950` | CDNA 4 |
| MI300 Series | MI325X, MI300X, MI300A | `gfx942` | CDNA 3 |
| MI200 Series | MI250X, MI250, MI210 | `gfx90a` | CDNA 2 |
| MI100 Series | MI100 | `gfx908` | CDNA |

### Radeon PRO 系列（工作站）

| 设备系列 | 具体型号 | LLVM Target | 架构 |
| --- | --- | --- | --- |
| AI PRO R9000 | R9700S, R9700, R9600D | `gfx1201` | RDNA 4 |
| PRO W7000 | W7900 Dual Slot, W7900, W7800 48GB, W7800 | `gfx1100` | RDNA 3 |
| PRO W7700 | W7700, V710 | `gfx1101` | RDNA 3 |
| **PRO W6000 / V 系列** | **W6800, V620** | **`gfx1030`** | **RDNA 2** |

最后一行值得单独说一句。很多流传的对照表都漏了它，但它同时又在 pip extras 表里出现了 `device-gfx1030`，看着像凭空多出来一个标签。实际上官方 7.14.0 的[支持硬件表](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html#amd-hardware-support)里确实有 Radeon PRO W6800 和 PRO V620，两张都是 gfx1030，也是官方支持列表里**唯一剩下的 RDNA 2 卡**。手里是 W6800 的不用慌，你是被支持的。

注意这里的边界：官方支持 gfx1030 指的是 W6800 和 V620 这两张专业卡，**不包括**同为 gfx1030 的消费级 RX 6900 XT / 6800 XT。想跑后者看第四节。

### Radeon RX 系列（消费级）

| 设备系列 | 具体型号 | LLVM Target | 架构 |
| --- | --- | --- | --- |
| RX 9000 | RX 9070 XT, 9070 GRE, 9070 | `gfx1201` | RDNA 4 |
| RX 9000 | RX 9060 XT LP, 9060 XT, 9060 | `gfx1200` | RDNA 4 |
| RX 7000 | RX 7900 XTX, 7900 XT, 7900 GRE | `gfx1100` | RDNA 3 |
| RX 7000 | RX 7800 XT, 7700 XT, 7700 XE, 7700 | `gfx1101` | RDNA 3 |
| RX 7000 | RX 7600 | `gfx1102` | RDNA 3 |

### Ryzen APU 系列（笔记本 / 移动端）

| 设备系列 | 具体型号 | LLVM Target | 架构 | iGPU |
| --- | --- | --- | --- | --- |
| AI Max PRO 400 | AI Max+ PRO 495, Max PRO 490/485 | `gfx1151` | RDNA 3.5 | Radeon 8060S |
| AI Max PRO 300 | AI Max+ PRO 395, Max PRO 390/385/380 | `gfx1151` | RDNA 3.5 | Radeon 8060S |
| AI Max 300 | AI Max+ 395, AI Max+ 392, AI Max+ 388, Max 390, Max 385 | `gfx1151` | RDNA 3.5 | Radeon 8060S / 8050S |
| AI PRO 400 | AI 9 HX PRO 475/470, AI 9 PRO 465, AI 7 PRO 450, AI 5 PRO 440 | `gfx1150` / `gfx1152` | RDNA 3.5 | Radeon 890M / 880M / 860M |
| AI 400 | AI 9 HX 475/470, AI 9 465, AI 7 450 | `gfx1150` / `gfx1152` | RDNA 3.5 | Radeon 890M / 880M / 860M |
| AI 400（7.14.0 新增） | AI 7 445, AI 5 435/430, AI 5 PRO 435 | `gfx1153` | RDNA 3.5 | 见下方说明 |
| AI 300 | AI 9 HX 375/370, AI 9 365, AI 7 350/345, AI 5 340/330 | `gfx1150` / `gfx1152` | RDNA 3.5 | Radeon 890M / 880M |
| Ryzen 200 | 9 270, 7 260/250, 5 240/230/220, 3 210 及 PRO 系列 | `gfx1103` | RDNA 3 | Radeon 780M / 760M / 740M |

`gfx1153` 是 7.14.0 新加的架构，对应 AI 7 445、AI 5 435/430、AI 5 PRO 435 这几颗。这点[7.14.0 release notes](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html) 和兼容性矩阵一致，可以放心用。

但它对应的 **iGPU 名字我没查准，所以表里留了空**：流传的对照表写的是 Radeon 860M / 840M，而 [TheRock 的 RELEASES.md](https://github.com/ROCm/TheRock/blob/main/RELEASES.md) 标的是「Radeon 820M iGPU」。官方兼容性矩阵把 820M 和 840M 都列进了 RDNA 3.5 的 iGPU，但那张表没给出 iGPU 名到 gfx 号的一对一映射，所以两种说法我都没法确认。既然拿不准，就不填一个看起来很确定的值。

**实操上这不影响你**——认 CPU 型号就够了，装包只看 gfx 号，不看 iGPU 叫什么。真要确认 iGPU 名称，以你机器上 `rocminfo` 的输出为准。

---

## 二、第二步：把 gfx 号变成安装命令

### device extras 速查

| LLVM Target | device extras 标签 |
| --- | --- |
| `gfx950` | `device-gfx950` |
| `gfx942` | `device-gfx942` |
| `gfx90a` | `device-gfx90a` |
| `gfx908` | `device-gfx908` |
| `gfx1201` | `device-gfx1201` |
| `gfx1200` | `device-gfx1200` |
| `gfx1100` / `gfx1101` / `gfx1102` | `device-gfx1100` / `device-gfx1101` / `device-gfx1102` |
| `gfx1030` | `device-gfx1030` |
| `gfx1151` | `device-gfx1151` |
| `gfx1150` | `device-gfx1150` |
| `gfx1152` | `device-gfx1152` |
| `gfx1153`（7.14.0 新增） | `device-gfx1153` |
| `gfx1103` | `device-gfx1103` |
| 全部架构 | `device-all` |

规律很简单：`gfx` 号前面加 `device-` 就是标签。换 GPU 只需要改这一处。

### 安装示例

以 gfx1151（Ryzen AI Max 系列）为例。**跑 PyTorch 只需要这一条命令**：

```bash
pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ \
  "torch[device-gfx1151]==2.12.0+rocm7.14.0" \
  "torchvision[device-gfx1151]==0.27.0+rocm7.14.0" \
  "torchaudio==2.11.0+rocm7.14.0"
```

三个细节：

- `torchaudio` **不带 extras**，别照着前两个的样子给它加。这里得先说清 host 和 device 这对词：**device 代码是编译成 GPU 机器码的那部分（也就是 kernel），只能在特定 gfx 架构上跑，换个架构就得重新编一份；host 代码跑在 CPU 上，负责分配显存、调度、把 kernel 发到卡上，和你用哪张卡无关。** multi-arch 拆包拆的就是这条线——host 那半所有人共用一份，device 那半按 gfx 号各编各的，extras 就是挑你要哪一份 device。被这么拆开的是 `rocm`、`torch`、`torchvision`；torchaudio 不在其中，说明它没有需要按架构编译的 device 代码，所以官方命令里它也是光着的。
- **版本号照抄，别自己按规律配。** 这三个版本号不是从 gfx 号推出来的，是[官方安装页](https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html)为「7.14.0 + 你选的 PyTorch 版本 + 你的架构」直接给出的组合。gfx1151 下面官方只给两组：

| torch | torchvision | torchaudio |
| --- | --- | --- |
| 2.12.0 | 0.27.0 | 2.11.0 |
| 2.11.0 | 0.26.0 | 2.11.0 |

  注意 torchaudio 2.11.0 同时对应 torch 2.12.0 和 2.11.0，它不跟着 torch 一起递增——这就是为什么不能自己推。而且**可选的 PyTorch 版本按架构和 OS 变化**：`device-all` 和 Instinct 那边还有一组 torch 2.10.0 / torchvision 0.25.0 / torchaudio 2.10.0，gfx1151 下面没有。换架构就回页面重选一次，别抄别人架构下的版本号。
- **不用先装 ROCm。** `torch` 本身就依赖 `rocm[libraries]`，`rocm-sdk-core`、`rocm-sdk-libraries`、`rocm-sdk-device-gfxXXXX` 会作为依赖一起拉下来。不少教程会让你先跑一条 `pip install "rocm[libraries,device-gfxXXXX]"`，纯 PyTorch 场景下那一步是多余的，下面单独说。

全文都写 `pip install`，和官方安装页一致（官方写的是更严谨的 `python -m pip install`，机器上有多个 Python 版本时用这个形式更保险）。想快的话把 `pip` 换成 [`uv pip`](https://github.com/astral-sh/uv) 就行，参数完全一样——ROCm 这套 wheel 动辄上 GB，uv 的并行下载和缓存在这里省的是分钟级的时间。代价是 uv 要另外装一次（`pip install uv`），所以下面的命令我都按最通用的 `pip` 写。

要做一个同时跑在多种卡上的镜像，extras 可以叠加：

```bash
# 一个 Dockerfile 同时给 MI300X 和 MI355X 用
pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ \
  "torch[device-gfx942,device-gfx950]" \
  "torchvision[device-gfx942,device-gfx950]" torchaudio
```

### 什么时候才需要单独装 `rocm[...]`

`rocm` 是个元包，方括号里每写一项就多拉一个组件：

| 写法 | 拉到的包 | 内容 |
| --- | --- | --- |
| `rocm` | `rocm-sdk-core` | SDK 核心，编译器和工具链 |
| `rocm[libraries]` | `rocm-sdk-libraries` | 架构无关的 host 侧库 |
| `rocm[device-gfxXXXX]` | `rocm-sdk-device-gfxXXXX` | 该架构的 device 代码 |
| `rocm[devel]` | `rocm-sdk-devel` | 开发工具，编译自己的算子时才要 |

**只跑 PyTorch 的话这条命令不用执行**，上面说了，torch 会自动把它们带下来。而且 [RELEASES.md](https://github.com/ROCm/TheRock/blob/main/RELEASES.md) 把这点写成了 WARNING 而不是普通提示：先装的 ROCm 版本万一和 torch wheel 要求的对不上，pip 会把它降级——白装一遍不说，还多一个出问题的环节。

真正需要它的是这三种情况：

1. **装 JAX。** 这是文档里唯一明写「必须先装」的场景——和 PyTorch 不同，JAX 的 wheel 不会自动拉 ROCm 依赖，不先装就是缺库。
2. **要用 ROCm SDK 本身**，不只是跑 PyTorch。比如用 `hipcc` 编译 HIP 代码、跑 `rocminfo` 查设备、拿 HIP 后端编 llama.cpp。这些工具在 `rocm-sdk-core` 里。
3. **编译自定义算子**，这时要加上 `devel`：

```bash
pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ \
  "rocm[libraries,devel,device-gfx1151]==7.14.0"
```

### 这里有个坑：不带 extras 等于空装

这是新机制带来的、旧文档里不会提醒你的问题。

这里说的“老办法”不是默认的 `pip install torch`，而是 AMD 以前的**分架构 wheel 仓库**。例如 ROCm 7.13 给 gfx1151 的官方命令把架构直接写在 URL 路径里：

```bash
python -m pip install --index-url https://repo.amd.com/rocm/whl/gfx1151/ \
  "torch==2.11.0+rocm7.13.0"
```

这个 URL 里的 `gfx1151` 限定了 pip 只能从 gfx1151 的仓库找包，因此其中的 `torch` wheel 已经打进了 gfx1151 的 device 代码；当时无需再写 `device-gfx1151`。换 GPU 时则要换整个 `--index-url`。这不是普通 PyPI 上裸跑 `pip install torch` 也会得到 GPU 包的意思；旧命令可在 AMD 的 [ROCm 7.13 PyTorch 安装页](https://rocm.docs.amd.com/en/7.13.0-preview/frameworks/pytorch/install.html)核对。

**换成 multi-arch 索引后，架构不再藏在 URL 里，`torch` 也被拆成了「host 代码」和「device 代码」两半；extras 才是决定要哪个 device 半边的开关。** 不给 extras，pip 不会报错，也不会挑一个默认架构，它就老老实实给你装一个只有 host 代码的 torch：

```bash
# 错的：能装上、能 import、跑不了 GPU
pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ torch
```

[TheRock #5289](https://github.com/ROCm/TheRock/issues/5289) 里把体积摊开写了，很能说明问题：

| 命令 | 装到什么 | 体积 |
| --- | --- | --- |
| `pip install torch` | 只有 host，**零 device 代码** | 约 431 MB |
| `pip install torch[device-gfx1100]` | host + gfx1100 的 device 代码 | 约 1.1 GB |
| `pip install torch[device-all]` | host + 该索引上所有架构的 device wheel | 约 5.5 GB（估算） |

**所以体积就是最快的自检手段**：装完发现 torch 相关的下载量只有 400 多 MB，不用往下查驱动了，你漏了 extras。想更直接一点，`pip freeze | grep amd-torch-device` 应该能看到 `amd-torch-device-gfxXXXX`，看不到就是空装。

引用这几个数字要说清边界。前两行是 #5289 对 nightly multi-arch 索引**当前行为**的描述，和下面「体积构成」那份 `pip freeze` 示例能互相印证。第三行的 5.5 GB 性质不同——它出现在那条 issue 的方案讨论里，是对「全部 16 个 device wheel 加起来有多少」的**估算**，那条 issue 至今仍是 open 状态。包集合是同一批，所以量级可以参考，但别当实测值。另外 #5289 原文写的是 `torch[all]`、`torch[gfx1100]`，我按当前的 `device-` 命名统一过了，点进去会看到两种写法。

还有两点：这些都是 nightly 上的数，我没在 7.14.0 发布版索引上实测过；「16 个架构」是那条 issue 写作时的数量，[RELEASES.md](https://github.com/ROCm/TheRock/blob/main/RELEASES.md) 现在已经列到 26 个 device 目标，所以 `device-all` 的真实体积应该比 5.5 GB 更大。都只看量级。

### 顺带说说体积构成

拆包之后装下来的东西具体长什么样，[RELEASES.md](https://github.com/ROCm/TheRock/blob/main/RELEASES.md) 给了一份 `pip freeze` 示例。一次单架构安装是这五项：

| 包 | 体积 | 性质 |
| --- | --- | --- |
| `rocm-sdk-core` | 约 700 MB | 跨 GPU 共用 |
| `rocm-sdk-libraries` | 约 100 MB | 跨 GPU 共用（架构无关的 host 侧库） |
| `torch` 本体 | 约 100 MB | 跨 GPU 共用（host 代码） |
| `rocm-sdk-device-gfxXXXX` | 约 50 MB | 只含该架构的 device 代码 |
| `amd-torch-device-gfxXXXX` | 约 50 MB | 只含该架构的 device 代码 |

这五项相加是 1 GB 出头，RELEASES.md 把这次安装的总量标成约 1.1 GB，和上面那张表里 `torch[device-gfx1100]` 那一行对得上——两个来源互相印证。

**结构上的重点是：约 900 MB 是跨 GPU 共用的，只有最后两项跟架构走。** 这也是 multi-arch 这次改造真正的好处——以前多架构镜像要塞好几套完整 SDK，现在共用部分只留一份。

但别拿「一个架构 50 MB」去乘架构数反推 `device-all`，对不上的。device 包的体积按架构差得很远，[#5289](https://github.com/ROCm/TheRock/issues/5289) 里同一次统计中 `amd-torch-device-gfx900` 是 54 MB，`amd-torch-device-gfx1100` 是 241 MB，差了四倍多。上面那个 5.5 GB 是按各架构实际大小堆出来的，不是均匀分摊的结果。

实操上的结论很简单：除非你真在做通用镜像，别无脑上 `device-all`，多出来的几个 G 全是你用不到的架构。

---

## 三、第三步：再查一次 OS，gfx 号推不出环境支持

这一节是我最想让人别跳过的。

前面两节的顺序会养成一个坏习惯：查到 gfx 号，装上包，开跑。**但 gfx 号是「编译目标」，不是「支持承诺」。** 你的 GPU 有 gfx 号、pip 有对应 extras、包也顺利装上了——这三件事加起来，依然推不出「你这台机器加这个系统是官方支持的组合」。

最典型的是 WSL。看下面这张表里这两组对比：

| GPU 类别 | Linux | Windows | WSL |
| --- | --- | --- | --- |
| Instinct (CDNA) | Ubuntu 22.04/24.04/26.04, RHEL 9/10, Debian 12/13, SLES 15/16 | ❌ | ❌ |
| Radeon AI PRO R9700S, R9600D (gfx1201) | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | ✅ |
| Radeon AI PRO R9700 (gfx1201) | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | ❌ |
| **Radeon RX 9070 (gfx1201)** | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | **✅** |
| **Radeon RX 9070 XT, 9070 GRE (gfx1201)** | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | **❌** |
| Radeon RX 9060 (gfx1200) | Ubuntu 22.04/24.04/26.04 | ✅ Windows 11 25H2 | ✅ |
| Radeon RX 9060 XT LP, 9060 XT (gfx1200) | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | ❌ |
| Radeon PRO W7900 / W7800 / RX 7900 (gfx1100) | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | ❌ |
| Radeon RX 7800/7700/7600 (gfx1101/1102) | Ubuntu 22.04/24.04/26.04, RHEL 9/10 | ✅ Windows 11 25H2 | ❌ |
| **Ryzen AI Max+ PRO 395, Max+ 395 (gfx1151)** | Ubuntu 26.04 / 24.04.4（HWE 内核） | ✅ Windows 11 25H2 | **✅** |
| **Ryzen AI Max 其他型号 (gfx1151)** | Ubuntu 26.04 / 24.04.4（HWE 内核） | ✅ Windows 11 25H2 | **❌** |
| Ryzen AI 9 HX (PRO) 475, 375 (gfx1150) | Ubuntu 26.04 / 24.04.4（HWE 内核） | ✅ Windows 11 25H2 | ✅ |
| Ryzen AI 其他型号 (gfx1150/gfx1152) | Ubuntu 26.04 / 24.04.4（HWE 内核） | ✅ Windows 11 25H2 | ❌ |
| Ryzen 200 (gfx1103) | Ubuntu 26.04 / 24.04.4（HWE 内核） | ✅ Windows 11 25H2 | ❌ |

**APU 那五行没有 RHEL，这不是我漏了。** [7.14.0 release notes](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html) 把操作系统支持分成三张表，APU 那张只有 Ubuntu（26.04 GA 内核 7.0、24.04.4 HWE 内核 6.17）和 Windows 11 25H2，根本没有 RHEL 行；正文也写得很直接——「adds support for RHEL 10.2 and RHEL 9.8 on AMD Instinct and Radeon GPUs」，只提 Instinct 和 Radeon。**RHEL 是独显的事，APU 上没有。** 顺带注意 APU 的 24.04.4 要求的是 HWE 内核而不是 GA 内核，独显那边是 GA 6.8，这一格不能照抄。

**「HWE 内核」这一格有具体门槛，Ryzen AI Max 的人尤其别跳过。** gfx1151 要靠两个 KFD 驱动补丁来修正队列创建和显存可用性检查，[RDNA 3.5 系统优化页](https://rocm.docs.amd.com/en/latest/reference/system-optimization/rdna3-5.html)给了最低版本：Ubuntu 24.04 HWE 至少 `6.17.0-19.19~24.04.2`，OEM 内核至少 `6.14.0-1018`，其他发行版至少 `6.18.4`。达不到的话官方的说法是 GPU 计算负载「可能起不来或者行为不可预测」。

**这个坑值得单独记一下，因为它的症状和第二节那个空装坑长得一模一样**——`torch.cuda.is_available()` 给你 False，但这次和 extras 一点关系没有。所以装之前先 `uname -r` 看一眼，别对着 pip 排查半天。

（那一页的分 ROCm 版本表最新只列到 7.11.0 / 7.12.0，**没有 7.14.0 的行**，所以我不说「7.14.0 在那张表上是绿的」。能说的是：内核补丁这个要求本身跟 ROCm 版本无关，而且它给的最低 HWE 版本和 release notes 里 APU 那张表的 HWE 6.17 对得上，两个来源不冲突。）

加粗那两组是重点：

- **RX 9070 有 WSL 支持，RX 9070 XT 没有。** 两张卡同为 gfx1201，装的是同一个 `device-gfx1201` 包。
- **AI Max+ 395 有 WSL 支持，同为 gfx1151 的其他 AI Max 型号没有。**

也就是说，**WSL 支持是按具体 SKU 走的，不是按架构走的**（SKU 指厂商实际拿出来卖的那一个具体型号，同一个架构下往往有好几个 SKU）。「我这卡和它同一个 gfx 号，它能跑 WSL，我应该也行」这个推理不成立。你在论坛上看到有人在 WSL 里跑通了 RX 9070，那不代表你的 9070 XT 也行——差一个后缀，结论就反了。

推广一点说：**GPU × OS × 后端，每个组合都有自己的支持表，别拿相邻组合的结论往你的场景上套。** 同理，Instinct 那一行 Windows 和 WSL 都是 ❌，不管你的 MI300X 上装了什么。

**一处我没写的**：上面这张 OS 表里没有 gfx1030 的行，因为兼容性矩阵那张静态表把 Radeon 各架构的 OS 行合并渲染了，读不出单独属于 gfx1030 的一行，我不想凭 RDNA 3 那几行往回推一个给你——这篇文章反对的就是这种推断。

手里是 W6800 或 V620 的，按这两个页面查：

- [7.14.0 release notes 的 AMD hardware support 表](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html#amd-hardware-support)：静态表格，不用点选择器，直接能看到 Radeon PRO W6800 和 Radeon PRO V620 两行，gfx1030 / RDNA 2。
- [兼容性矩阵](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)：查操作系统支持。注意得先把 Device family 切到 AMD Radeon™，默认的 Instinct 视图里没有这两张卡。

再提一个值得你自己去核的点：[Linux system requirements](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/reference/system-requirements.html) 里有逐型号的操作系统脚注，而 W6800 和 V620 挂的**不是同一条**——同一个 gfx1030，这两张卡的发行版支持大概率不一样。但那一页顶部挂着「已迁移」提示、版本号比矩阵旧一档，所以我不照抄它的结论，建议你去矩阵里按自己的型号复核一遍。

---

## 四、表里没有你的卡怎么办

前面所有表都是**官方支持范围**。没在表里，不等于跑不起来，但要清楚代价。按从省事到折腾排：

**1. 先确认是不是真没有。** 官方支持列表在变，7.14.0 就新加了 gfx1153。以[release notes 的支持硬件表](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html#amd-hardware-support)和[兼容性矩阵](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)为准，别以为几个月前查过就是定论。

**2. 试 TheRock 的 nightly 索引。** ROCm 的上游构建项目 [TheRock](https://github.com/ROCm/TheRock/blob/main/RELEASES.md) 的 device extras 列表比官方发布版宽得多——官方 7.14.0 覆盖 15 个架构，TheRock 现在列了 26 个。多出来的这些卡是这样对应的：

| device extras | 覆盖的卡 |
| --- | --- |
| `device-gfx1031` / `device-gfx1032` / `device-gfx1034` | RX 6750 XT / 6700 XT，RX 6600 XT / 6600 / W6600，RX 6500 XT |
| `device-gfx1030`（这个标签官方也有） | RX 6900 XT / 6800 XT——官方那份 gfx1030 只覆盖 W6800 / V620 两张专业卡，消费级 RX 6000 得走这条 |
| `device-gfx1035` / `device-gfx1036` / `device-gfx1033` | Radeon 680M、Raphael iGPU、Van Gogh iGPU |
| `device-gfx1010` / `device-gfx1011` / `device-gfx1012` | RX 5700 / 5700 XT，PRO V520，PRO W5500 |
| `device-gfx906` / `device-gfx900` | MI60 / MI50 / Radeon VII / Radeon Pro VII，MI25 |

用法和第二节一样，只是索引换成 nightly 的：

```bash
pip install --index-url https://rocm.nightlies.amd.com/whl-multi-arch/ \
  "torch[device-gfx1031]" "torchvision[device-gfx1031]" torchaudio
```

*上游项目路径，我没有在这些老卡上逐个实测过。* **代价要说清楚**：这是 nightly，没有 SLA（AMD 不对它做任何可用性承诺），版本天天动，今天通的下周可能回归；出问题只能去 GitHub issue 问，不走官方支持渠道。适合个人机器折腾和验证想法，别拿它撑生产。还要分清一件事——「有 extras、能装上」只说明有这个架构的编译产物，不等于全部算子库和上层框架都在这张卡上验证过，实际能不能跑通你的模型得自己试。

**3. 换后端。** 只做推理、不碰 PyTorch 训练的话，跳出 ROCm 反而更省事：llama.cpp 的 Vulkan 后端不依赖 ROCm 支持表，Windows 上还有 DirectML 一条路。*这两条都不在 ROCm 官方支持范围内，我也没有在上面列的老卡上实测过，只能说是方向。* 代价是能力窄——基本只覆盖推理，训练、自定义算子、需要 HIP 的项目都用不上，性能也别拿官方 ROCm 路径的数字去期待。RDNA 1/2 的消费卡想在 Windows 上跑本地推理，值得先试这条再回头硬凑 ROCm。

**4. 最后才考虑换硬件。** 真要换，按 gfx 号选：新平台优先 gfx1201（RDNA 4）或 gfx1151（Ryzen AI Max，统一内存对大模型友好），这两个在官方表里覆盖最全，WSL 也有型号支持。

顺序别倒过来——绝大多数「我的卡跑不了 ROCm」最后都是装漏了 extras 或者系统组合没对上，不是硬件不行。

---

## 总结

三步，别跳步：

1. **查 gfx 号**（第一节的四张表）→ 决定 extras 标签是什么。
2. **拼安装命令**（第二节）→ `--index-url https://repo.amd.com/rocm/whl-multi-arch/` 加 `[device-gfxXXXX]`，**extras 一定要带**，装完看体积或者 `pip freeze | grep amd-torch-device` 验一下。
3. **核对 OS**（第三节）→ 按你的**具体型号**查，尤其 WSL，别按 gfx 号推。

一句话记住：**gfx 号解决「装什么」，型号 + OS 解决「能不能跑」，这两个问题不能互相替代。**

来源都是可点开核对的：

- [7.14.0 release notes 的 AMD hardware support 表](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html#amd-hardware-support)（静态支持硬件表，gfx1030 那两张卡在这；同一页还有分 Instinct / Radeon / APU 的三张 OS 表）
- [ROCm 7.14.0 兼容性矩阵](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)（按型号查 OS 和 WSL 的选择器在这，记得先切 Device family）
- [PyTorch on ROCm 安装页](https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html)（三件套版本组合的一手来源，按架构和 OS 分别给）
- [ROCm 7.13 安装页](https://rocm.docs.amd.com/en/7.13.0-preview/frameworks/pytorch/install.html)（旧的分架构 index URL 长什么样，可对照）
- [TheRock RELEASES.md](https://github.com/ROCm/TheRock/blob/main/RELEASES.md)（完整 device extras 列表和 `pip freeze` 体积示例，含官方发布版没有的架构）
- [TheRock #5289](https://github.com/ROCm/TheRock/issues/5289)（空装那几个体积数字的出处，注意它是一条 open 的提案）

以上。

哪张卡、什么系统、卡在哪一步，评论区说具体型号，别只说「RX 7000」——这篇文章的重点就是型号和架构不能互相代替。
