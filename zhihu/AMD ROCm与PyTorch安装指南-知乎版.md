# 从 gfx 到第一个 GPU Tensor：AMD ROCm 7.14 + PyTorch 安装指南

本文包括先按完整型号查 gfx 与操作系统支持，再装PyTorch，最后用一次真实张量计算确认 GPU 可用。

我以 ROCm Core SDK 7.14.0 与 PyTorch 2.12.0 为主线，重点写原生 Linux，也保留 Windows、WSL2、Docker 和官方表外显卡的分流。硬件、系统和版本信息已按 2026-08-04 的官方页面复核；发布前仍建议读者打开文末链接按自己的完整型号再查一次。

本文的数据来源是：AMD release notes 与兼容性矩阵用于硬件和系统支持；PyTorch 安装页用于版本与安装命令；TheRock 用于 nightly 目标。PyTorch 安装页是滚动页面，使用前请确认页标题仍为 ROCm 7.14.0。TheRock #5289 是开放提案，只用来说明包体积量级，不当作发布版实测。

## ROCm安装的所有路径

ROCm 安装不是只分显卡型号。完整判断条件是具体 SKU × 操作系统 × 内核或驱动 × 框架版本。下面先列出全部常见路径，再决定该复制哪一段命令。

| 路径 | 状态与验证 | 适合什么 | 主要代价或边界 |
| --- | --- | --- | --- |
| 原生 Linux + pip | 官方；须在兼容性矩阵核对具体型号、发行版、内核与驱动 | 日常 PyTorch 开发、训练、编译扩展 | 需要先让内核驱动和设备权限正常 |
| 原生 Linux + Docker | 官方；AMD 提供 ROCm PyTorch 镜像 | 希望环境可复现，或不想把 Python 依赖装进系统 | 容器不能替代宿主机驱动，宿主机仍须识别 GPU |
| 原生 Windows + pip 或 tarball | 官方；仅限矩阵中列出的具体 SKU 与 Windows 版本 | 在 Windows 上运行受支持的 Python 工作负载 | 不要套用 Linux 的 amdgpu 驱动命令 |
| WSL2 + Ubuntu | 官方但按具体 SKU 验证 | 需要 Windows 桌面与 Linux Python 工作流共存 | Windows 主机驱动、WSL 发行版和 GPU 型号缺一不可 |
| TheRock nightly | AMD 上游 nightly；有编译产物不等于发布版验证 | 官方表外显卡的个人验证 | 没有发布版 SLA，版本可能回归 |
| DirectML 或 Vulkan | 替代后端，不属于 ROCm 官方 PyTorch 路径 | Windows 或老卡上的本地推理，尤其是 llama.cpp 一类程序 | 训练、自定义 HIP 算子与算子覆盖不能按 ROCm 预期 |

我的建议是：官方支持的设备优先在原生 Linux 中通过 pip 安装；需要隔离依赖时，再使用官方 Docker 镜像。先核对显卡型号、系统、内核、驱动和 PyTorch 安装参数；官方路径仍无法满足需求，再尝试 TheRock 等社区构建，最后才考虑更换硬件。即使 gfx 相同，Windows、WSL2 与 Linux 的支持状态也可能不同。

## 先查完整 gfx：device extra 从这里取值

可参考过往 [文章](https://zhuanlan.zhihu.com/p/2067663713826612548) 找到 gfx代号，再到后面的 device extra 表中取安装标签。

### gfx 到 device extra

安装命令中写的是 device extra。两者一一对应，在后面会用到：

| gfx | device extra |
| --- | --- |
| gfx950 | device-gfx950 |
| gfx942 | device-gfx942 |
| gfx90a | device-gfx90a |
| gfx908 | device-gfx908 |
| gfx1201 | device-gfx1201 |
| gfx1200 | device-gfx1200 |
| gfx1100 | device-gfx1100 |
| gfx1101 | device-gfx1101 |
| gfx1102 | device-gfx1102 |
| gfx1030 | device-gfx1030 |
| gfx1151 | device-gfx1151 |
| gfx1150 | device-gfx1150 |
| gfx1152 | device-gfx1152 |
| gfx1153 | device-gfx1153 |
| gfx1103 | device-gfx1103 |
| 所有发布版架构 | device-all |

## 安装Rocm

光靠gfx号不能推出安装的环境要求, 需要根据下面这张表查询。
下表按 ROCm 7.14.0 分支的[安装选择器](https://rocm.docs.amd.com/en/docs-7.14.0/install/rocm.html)整理。最后仍要在 [兼容性矩阵](https://rocm.docs.amd.com/en/docs-7.14.0/compatibility/compatibility-matrix.html) 选择完整 SKU 复核。完整型号决定设备支持，操作系统、驱动和 WSL2 状态还要单独核对。

安装rocm过程在[安装选择器](https://rocm.docs.amd.com/en/docs-7.14.0/install/rocm.html)查询。

| GPU 或 APU | Linux | Windows | WSL2 |
| --- | --- | --- | --- |
| Instinct 系列 | 按 MI SKU 查询；各型号的发行版与内核不同 | ❌ | ❌ |
| Radeon AI PRO R9700S、R9700，gfx1201 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ✅ |
| Radeon AI PRO R9600D，gfx1201 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ❌ |
| Radeon RX 9070 XT、9070 GRE、9070，gfx1201 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ✅ |
| Radeon RX 9060 XT、RX 9060，gfx1200 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ✅ |
| Radeon RX 9060 XT LP，gfx1200 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ❌ |
| gfx1100 的 PRO W7900、W7800 与 RX 7900 系列 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ✅ |
| PRO W7700、RX 7800 XT，gfx1101 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ✅ |
| RX 7700 XT、RX 7700 XE、RX 7700，gfx1101 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ❌ |
| PRO V710，gfx1101 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ❌ | ❌ |
| RX 7600，gfx1102 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ✅ Windows 11 25H2 | ❌ |
| PRO W6800、PRO V620，gfx1030 | Ubuntu 22.04.5、24.04.4、26.04；RHEL 9.8、10.2 | ❌ | ❌ |
| 全部官方 gfx1151 Ryzen AI Max 型号 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ✅ |
| Ryzen AI 7 445、AI 5 435、430、AI 5 PRO 435，gfx1153 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ✅ |
| Ryzen AI 9 HX PRO 475、470；AI 9 PRO 465；AI 9 HX 475、470；AI 9 465、AI 9 HX 375、370、AI 9 365，gfx1150 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ✅ |
| Ryzen AI 9 HX PRO 375、370，gfx1150 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ❌ |
| Ryzen AI 7 PRO 450、AI 5 PRO 440、AI 7 450，gfx1152 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ✅ |
| Ryzen AI 7 PRO 350、AI 5 PRO 340、AI 7 350、345、AI 5 340、330，gfx1152 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ❌ |
| Ryzen 200，gfx1103 | Ubuntu 26.04；Ubuntu 24.04.4 HWE 6.17 | ✅ Windows 11 25H2 | ❌ |

注意：

1. R9700 与 R9600D 同为 gfx1201，WSL2 结论不同。
2. RX 9060 XT 与 RX 9060 XT LP 同为 gfx1200，WSL2 结论不同。
3. Ryzen AI 9 HX PRO 375 与 475 同为 gfx1150，WSL2 结论不同。

## 安装PyTorch前

一个 ROCm PyTorch 环境至少有四层：

1. 具体 GPU 和操作系统组合属于支持范围；
2. 内核驱动已创建可计算的 GPU 设备；
3. Python 环境安装了与 gfx 对应的 device 代码；
4. PyTorch 的版本组合与 Python 版本匹配。
   
pip 安装成功，只代表 Python 包和部分依赖已装好，不代表 ROCm 已经能使用 GPU。
我会先在 Linux 终端收集这四项：

    lspci -nn | grep -Ei 'vga|3d|display'
    cat /etc/os-release
    uname -r
    python3 --version
    
上面的 gfx 表决定安装标签，OS 表决定环境边界。这四项只能确认型号、系统、内核与 Python 版本；驱动装好后，再用 rocminfo 确认 ROCm 是否识别到计算设备，再到 [PyTorch 安装页](https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html) 选对应版本。

### 第二步：创建隔离的 Python 环境

ROCm 7.14.0 的官方 PyTorch 页面支持 Python 3.11 到 3.14。下面使用系统 python3；如果它是 Python 3.10 或更早版本，我建议换用受支持的解释器或 Docker。

Ubuntu 先补齐虚拟环境与运行库：

    sudo apt update
    sudo apt install python3-venv libatomic1 libquadmath0

再创建项目专用环境：

    python3 --version
    mkdir -p ~/venvs/pytorch-rocm
    cd ~/venvs/pytorch-rocm
    python3 -m venv .venv
    . .venv/bin/activate
    python -m pip install --upgrade pip

看到 .venv 且 python --version 在 3.11 到 3.14 范围内，就可以继续。以后使用前先运行 . .venv/bin/activate。

### 第三步：安装与 gfx 匹配的 PyTorch

ROCm 7.14.0 的 Python 仓库采用 multi-arch。索引地址不再区分 gfx，device-gfxXXXX extra 才决定下载哪份 GPU kernel。

我用 gfx1100 演示。将两处 gfx1100 换成前表查到的值，例如 gfx1201、gfx1151 或 gfx90a。

    python -m pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ \
        'torch[device-gfx1100]==2.12.0+rocm7.14.0' \
        'torchvision[device-gfx1100]==0.27.0+rocm7.14.0' \
        'torchaudio==2.11.0+rocm7.14.0'

这三个版本号必须成组使用，不要根据包名或 gfx 自行凑版本。

下表来自官方安装页。较旧组合不一定覆盖每个 gfx 和操作系统，最终以安装页选择器为准。

| torch | torchvision | torchaudio | 使用边界 |
| --- | --- | --- | --- |
| 2.12.0 | 0.27.0 | 2.11.0 | 本文默认示例，优先选官方当前组合 |
| 2.11.0 | 0.26.0 | 2.11.0 | 仅在安装页为当前 gfx 与 OS 列出时使用 |
| 2.10.0 | 0.25.0 | 2.10.0 | 较早的 ROCm 7.14 组合，仍须由安装页确认 |

我建议只记住三点：

- torch 与 torchvision 必须带 device-gfxXXXX，才能拉到对应架构的 device 代码。
- torchaudio 不带 device extra。
- 不要先自行安装另一套 rocm Python 元包。先完成安装页要求的宿主机前置条件，再让这条官方命令解析依赖。

### device-all 什么时候才值得装

单机开发选单一 gfx。下表来自 [TheRock #5289](https://github.com/ROCm/TheRock/issues/5289) 的 nightly 提案，只用于理解包体积。

| #5289 的 nightly 快照 | 统计口径 | 参考体积 | 不能推出什么 |
| --- | --- | --- | --- |
| torch，不带 extra | host-only torch 入口包 | 约 431 MB | 不是三件套总下载量 |
| torch[gfx1100] | host 加 gfx1100 和 gfx11 device wheel | 约 1.1 GB | 不是当前发布版 device-gfx 写法 |
| torch[all] | 当时全部 16 个 device wheel | 约 5.5 GB | 不是当前发布版实测 |

## 用两个检查确认不是空装

安装后，我先在 Linux 环境确认 device 包存在：

    python -m pip freeze | grep -E 'amd-torch-device|rocm-sdk-device'

输出应包含目标 gfx，例如 amd-torch-device-gfx1100。再检查 ROCm 实际报告的目标：

    rocminfo | grep -oE 'gfx[0-9a-z]+' | sort -u

笔记本、APU 加独显和虚拟化环境可能列出多个 gfx。不要猜，确认预期 GPU 的 gfx 已出现。

最后运行一次最小张量计算。它验证 GPU 执行，不代表性能测试：

    python - <<'PY'
    import torch

    print('torch:', torch.__version__)
    print('HIP:', torch.version.hip)
    print('available:', torch.cuda.is_available())

    if not torch.cuda.is_available():
        raise SystemExit('ROCm device is unavailable')

    print('device:', torch.cuda.get_device_name(0))
    x = torch.randn((2048, 2048), device='cuda')
    y = x @ x
    torch.cuda.synchronize()
    print('result:', y.device, y.shape, float(y[0, 0]))
    PY

我只看三个结果：

1. available 显示 True；
2. HIP 显示非空版本；
3. result 的设备为 cuda:0，且矩阵乘法正常返回。

ROCm 沿用 torch.cuda 和 cuda:0 以兼容 PyTorch API，这不表示程序在调用 NVIDIA GPU。

## Docker：想隔离 Python 依赖时的官方路径

Docker 只隔离用户态依赖，不会安装宿主机驱动。我只在原生 Linux 已能看到 /dev/kfd 与 /dev/dri 时使用它。

以下镜像来自官方 PyTorch 安装页，组合为 ROCm 7.14、Ubuntu 24.04、Python 3.12 与 PyTorch 2.12.0：

    docker pull rocm/pytorch:rocm7.14_ubuntu24.04_py3.12_pytorch_release_2.12.0

按官方示例启动：

    docker run --rm -it \
        --device=/dev/kfd \
        --device=/dev/dri \
        --network=host \
        --group-add=video \
        --ipc=host \
        --cap-add=SYS_PTRACE \
        --security-opt seccomp=unconfined \
        rocm/pytorch:rocm7.14_ubuntu24.04_py3.12_pytorch_release_2.12.0 bash

进入容器后先做最小检查：

    python -c 'import torch; print(torch.__version__); print(torch.cuda.is_available())'

## Windows 和 WSL2：同一个 gfx 也要重新核对

### 原生 Windows

命令如下：
    py -3.12 -m venv .venv
    .\.venv\Scripts\Activate.ps1
    python -m pip install --upgrade pip
    python -m pip install --index-url https://repo.amd.com/rocm/whl-multi-arch/ 'torch[device-gfx1201]==2.12.0+rocm7.14.0' 'torchvision[device-gfx1201]==0.27.0+rocm7.14.0' 'torchaudio==2.11.0+rocm7.14.0'

### WSL2

我把 WSL2 当成 Windows 主机和 Ubuntu 客体两套环境，按下面顺序处理：

1. 先在兼容性矩阵确认完整 GPU SKU 的 WSL2 支持；
2. 在 Windows 主机安装对应 AMD 驱动，并使用官方列出的 Ubuntu WSL2 版本；
3. 按 [ROCm 7.14.0 安装页](https://rocm.docs.amd.com/en/docs-7.14.0/install/rocm.html) 的 WSL2 选择器完成前置步骤；需要 ROCDXG 时，再按 [librocdxg Quickstart](https://github.com/ROCm/librocdxg) 操作。

不要在 WSL2 内安装 amdgpu DKMS。WSL 的 GPU 访问依赖 Windows 主机驱动和虚拟化链路，不能混用原生 Linux 指南。

## 官方表外的显卡，先降低目标再选方案

型号不在 ROCm 7.14.0 官方表时，我会先确认不是在看旧页面。确认无误后可以尝试下面的路径，但我不会把它们称为官方支持。

### TheRock nightly

[TheRock](https://github.com/ROCm/TheRock) 的上游 nightly 提供更多 gfx 目标。它适合新的虚拟环境验证，不适合替代稳定发布版。

下表来自 [TheRock RELEASES.md](https://github.com/ROCm/TheRock/blob/54d14392b27167b862bf5747f1d8cd1b13a4b23c/RELEASES.md)，这是 2026-08-04 复核时的提交。它只说明存在编译产物，不承诺完整算子、框架或模型验证。

| nightly device extra | 覆盖的卡或 iGPU | 边界 |
| --- | --- | --- |
| device-gfx1031 | RX 6750 XT、RX 6700 XT | RDNA 2，非发布版支持 |
| device-gfx1032 | RX 6600 XT、RX 6600、PRO W6600 | RDNA 2，非发布版支持 |
| device-gfx1033 | Van Gogh iGPU | 非发布版支持 |
| device-gfx1034 | RX 6500 XT | RDNA 2，非发布版支持 |
| device-gfx1035 | Radeon 680M iGPU | 非发布版支持 |
| device-gfx1036 | Raphael iGPU | 非发布版支持 |
| device-gfx1030 | RX 6900 XT、RX 6800 XT | 发布版官方 gfx1030 只覆盖 PRO W6800、PRO V620 |
| device-gfx1010 | RX 5700、RX 5700 XT | RDNA 1，非发布版支持 |
| device-gfx1011 | PRO V520 | RDNA 1，非发布版支持 |
| device-gfx1012 | PRO W5500 | RDNA 1，非发布版支持 |
| device-gfx906 | MI60、MI50、Radeon VII、Radeon Pro VII | 较旧卡，非发布版支持 |
| device-gfx900 | MI25 | 较旧卡，非发布版支持 |

    python -m pip install --index-url https://rocm.nightlies.amd.com/whl-multi-arch/ \
        'torch[device-gfx1031]' \
        'torchvision[device-gfx1031]' \
        torchaudio

请在新的虚拟环境中尝试，不要覆盖能工作的官方环境。能下载只说明存在该 gfx 编译产物；nightly 没有发布版 SLA，也可能回归。

### DirectML、Vulkan 与 ZLUDA

- Windows 上只做推理时，我会评估 [PyTorch with DirectML](https://github.com/microsoft/DirectML)。它不是 ROCm PyTorch 后端，算子、性能和训练能力要按项目验证。
- llama.cpp 等程序可评估 [Vulkan 后端](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)。它适合本地推理，不替代 HIP 自定义算子或 PyTorch 训练。
- ZLUDA 与项目专用 fork 是社区兼容层，不是普通 PyTorch 用户安装 ROCm 的快捷方式。

只有项目确实需要长期训练、定制算子和官方支持时，我才会最后考虑硬件升级。购买仍按完整 SKU 和目标系统判断，不只看 gfx。

## 常见问题：先按环境判断根因

| 适用环境 | 现象 | 更可能的根因 | 最小处理方式 |
| --- | --- | --- | --- |
| 原生 Linux | torch.cuda.is_available 为 False | 漏装 device extra，或设备权限异常 | 查 device 包；再查 /dev/kfd 与 render、video 组 |
| 原生 Linux | 没有 /dev/kfd | 驱动、内核、Secure Boot 或系统组合不对 | 回到宿主机驱动与兼容性矩阵，不要重装 Python 包 |
| Linux、Windows、WSL2 | 出现 no kernel image 一类错误 | 安装了错误 gfx 的 device 包 | 对照当前设备目标与 device-gfxXXXX，在新 venv 重装 |
| Windows、WSL2 | torch.cuda.is_available 为 False | GPU SKU、Windows 驱动、WSL 发行版或 ROCDXG 链路不匹配 | 按 OS 表逐项核对，不以 /dev/kfd 作为判断条件 |
| 原生 Linux | 导入时提示缺少 libnuma | 缺少系统 NUMA 库 | Ubuntu 执行 sudo apt install libnuma1，再重新打开 venv |
| 所有环境 | pip 找不到 wheel | Python 或版本组合不支持 | 确认 Python 为 3.11 到 3.14，并按官方页面成组安装 |

原生 Linux 排查时，我会收集完整型号、系统版本、uname -r、rocminfo 的 gfx 输出和 device 包名。Windows 与 WSL2 则改查完整型号、Windows 驱动、WSL 发行版和 ROCDXG 状态。

## 可选优化：只在基础验证成功之后再碰

基础张量测试通过后，RX 7900、RX 7800 XT 或 Ryzen AI Max 跑 batch size 不小于 8 的 LLM FP16 解码时，官方提示 PyTorch 2.14 前可尝试 hipBLASLt。

可在一次单独的测试运行中设置：

    export TORCH_BLAS_PREFER_HIPBLASLT=1
    python your_script.py

这不是通用加速开关。我只会在相同模型、batch size、输入长度、精度和预热条件下比较端到端吞吐与显存；无改善或不兼容就回退：

    unset TORCH_BLAS_PREFER_HIPBLASLT

## 小结

我把流程压缩成四步：

1. 用完整型号确认 gfx，并在对应操作系统的矩阵中确认支持状态；
2. 原生 Linux 先让驱动、内核和 /dev/kfd 正常工作；Windows 与 WSL2 则先完成主机驱动链路；
3. 在隔离 venv 中安装带 device-gfxXXXX 的官方 PyTorch 三件套；
4. 原生 Linux 用 device 包、rocminfo 和矩阵乘法验证；Windows 与 WSL2 用主机链路加 PyTorch 验证。

这篇的目标是避免装出只能 import、却不能使用 AMD GPU 的 PyTorch。后续跑模型、训练或部署时，我会先按显存、精度和工作负载选量化、offload、镜像或推理框架，再判断是否需要硬件升级。

## 参考资料

- [ROCm 7.14.0 Release Notes 与官方硬件支持表](https://rocm.docs.amd.com/en/docs-7.14.0/about/release-notes.html)
- [ROCm 7.14.0 兼容性矩阵](https://rocm.docs.amd.com/en/docs-7.14.0/compatibility/compatibility-matrix.html)
- [安装 AMD ROCm 7.14.0](https://rocm.docs.amd.com/en/docs-7.14.0/install/rocm.html)
- [ROCm 7.14.0 的 PyTorch 安装页，滚动页面](https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html)
- [TheRock supported GPUs，2026-08-04 快照](https://github.com/ROCm/TheRock/blob/54d14392b27167b862bf5747f1d8cd1b13a4b23c/SUPPORTED_GPUS.md)
- [TheRock RELEASES.md，2026-08-04 快照](https://github.com/ROCm/TheRock/blob/54d14392b27167b862bf5747f1d8cd1b13a4b23c/RELEASES.md)
- [TheRock #5289](https://github.com/ROCm/TheRock/issues/5289)
## 名词解释
- SKU（Stock Keeping Unit）是厂商实际拿出来卖的那一个具体型号，比架构更细一层。同一个架构下有好几个 SKU：RX 9070 和 RX 9070 XT 都是 gfx1201、装同一个 device-gfx1201 包，但这两个是不同的 SKU——而 ROCm 的 WSL 按 SKU支持 ，9070 有、9070 XT 没有。用rocm-smi --showproductname查询，其中Card SKU:xxxx，是 AMD 板卡的内部编号。
