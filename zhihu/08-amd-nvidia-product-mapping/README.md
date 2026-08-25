# AI 教程常写 NVIDIA，换成 AMD 该看哪张？A/N 卡产品线与候选型号

打开一篇本地大模型、AI 绘图或训练教程，硬件建议经常是这样写的：5070 Ti 够用，4090 更稳，训练则要上 H100。

这些 NVIDIA 型号已经成了 AI 硬件的常用坐标。但如果你手里是 A 卡，问题马上就来了：RX 9070 XT 应该去参考哪类产品？W7900 能不能参考 RTX 6000 Ada 的使用场景？Instinct MI 系列又该怎么理解？

这篇文章先回答“去 AMD 这边看谁”。它是一份产品导航，不是性能对标：消费级按上市周期和市场位置找候选，工作站看专业产品与显存容量，数据中心采用 AMD 官方竞争参照，统一内存设备只比较它们解决的容量问题。真正的速度还要回到具体工作负载。

![AMD 与 NVIDIA 四类产品线总览](images/01-product-lines.png)

## 教程写 5070 Ti，A 卡先看哪张

消费级显卡最容易建立坐标，也最容易被误读成性能排名。下面四组只表示市场位置候选。

| NVIDIA 坐标 | NVIDIA 显存 | AMD 候选 | AMD 显存 | 为什么放在一组 |
|---|---:|---|---:|---|
| GeForce RTX 5070 Ti | 16GB GDDR7 | Radeon RX 9070 XT | 16GB GDDR6 | 同属 2025 高端消费级产品，容量相同 |
| GeForce RTX 5070 | 12GB GDDR7 | Radeon RX 9070 | 16GB GDDR6 | 同一周上市，美国官方起始价格同为 549 美元 |
| GeForce RTX 5060 Ti | 8GB / 16GB GDDR7 | Radeon RX 9060 XT | 8GB / 16GB GDDR6 | 同属 2025 主流产品家族，均有两种容量 |
| GeForce RTX 4080 | 16GB GDDR6X | Radeon RX 7900 XTX | 24GB GDDR6 | 同属 2022 高端 4K 产品周期，但各自产品栈级别并非严格相同 |

![GeForce RTX 与 Radeon RX 候选位置](images/02-consumer-map.png)

RX 9070 和 RTX 5070 是这里上市时间与美国官方起始价格最接近的一组，但显存已经不同：前者是 16GB，后者是 12GB。RX 9060 XT 和 RTX 5060 Ti 则都分 8GB、16GB 两个版本，购买或阅读教程时不能只看型号。

对本地大模型来说，模型权重、KV Cache 和运行时空间通常都要占显存。图像、视频生成还要给中间张量和分辨率、batch 带来的额外占用留空间。因此 RX 7900 XTX 的 24GB 与 RTX 4080 的 16GB 会给出不同的容量余量，但显存更大不自动代表生成更快。

找到候选卡之后，还要检查教程真正依赖什么。如果脚本调用 CUDA 专用库或算子，不能只替换显卡名；需要确认目标应用及所需算子，在当前操作系统、GPU 和软件版本上是否有 ROCm、Vulkan 或其他可用实现。

## 32GB 和 48GB，再看工作站卡

如果教程已经写到 RTX PRO，AMD 侧应优先看 Radeon PRO / AI PRO，而不是只在 Radeon RX 游戏卡里找。

| NVIDIA 坐标 | 显存 | AMD 候选 | 显存 | 产品关系 |
|---|---:|---|---:|---|
| RTX PRO 4500 Blackwell Workstation Edition | 32GB GDDR7 ECC | Radeon AI PRO R9700 | 32GB GDDR6 | 同属 2025 专业工作站周期，本地 AI 与专业创作用途有交集 |
| RTX 6000 Ada Generation | 48GB GDDR6 ECC | Radeon PRO W7900 | 48GB GDDR6 ECC | 同时代旗舰工作站产品，容量和目标用途接近，不按价格配对 |

![RTX PRO 与 Radeon PRO 候选位置](images/03-workstation-map.png)

这两张 AMD 卡不是只存在于规格表里。我们在一台 4×R9700 宿主上锁定单卡，用 ROCm 7.2.0、PyTorch 2.9.1 和 Transformers 5.14.1 跑通了 Qwen2.5-0.5B-Instruct；另一组独立实验则用单卡 llama.cpp ROCm / HIP 跑过 Qwen3.6-35B-A3B Q4_K_M。W7900 这边，我们用 llama.cpp Vulkan 跑通 Qwen3.8 27B Q4_K_M，开启 MTP 后约占 19.2 GiB，65 / 65 个模型层全部放进 GPU。

这些观察说明的是容量怎样落到实际任务，不是 R9700 或 W7900 与 NVIDIA 候选的速度关系。32B Q4 权重常见约 20GB，放进 32GB 卡后还能给 KV Cache 和 workspace 留空间；70B Q4 权重常见约 40—45GB，进入 48GB 单卡后余量已经很紧，实际占用仍会随上下文、精度和框架变化。

R9700 还明确面向本地开发、模型微调和多 GPU 工作站。但多张 32GB 卡不会自动变成一个对程序透明的 64GB 显存池。普通数据并行通常在每张卡复制完整模型，主要扩大吞吐或全局 batch；如果要让单卡放不下的模型或训练状态跨卡，需要张量并行、流水线并行、FSDP / ZeRO 等分片方式。互联带宽、通信库和并行策略也会进入选型。

## A100、H100、B200，对面是 Instinct 哪一代

数据中心部分不是按价格排列。下面四组是 AMD 官方产品材料实际采用的 NVIDIA 竞争参照。

| NVIDIA 竞争参照 | 代表容量 | AMD Instinct | 容量 | 关系 |
|---|---:|---|---:|---|
| A100 80GB SXM | 80GB HBM2e | MI250X | 2×64GB HBM2e | AMD 发布材料的直接比较对象 |
| H100 SXM | 80GB HBM3 | MI300X | 192GB HBM3 | AMD 官方比较过 OAM / HGX 配置 |
| H200 SXM | 141GB HBM3e | MI325X | 256GB HBM3E | 两者都是原架构上的高容量刷新产品 |
| B200 SXM / HGX B200 | 180GB HBM3e / GPU | MI355X | 288GB HBM3E | AMD 官方采用的单 GPU 与 8-GPU 平台竞争参照 |

![AMD Instinct 与 NVIDIA 数据中心 GPU 竞争参照](images/04-datacenter-map.png)

这一部分面向数据中心训练、推理服务和多卡集群。容量决定模型和运行状态能不能放下，计算精度支持会影响 FP8、FP4 等训练或推理路径，内存带宽影响数据在 GPU 内移动的上限，GPU 间互联影响多卡扩展，软件框架、通信库和优化 kernel 则决定峰值规格能不能在真实任务中兑现。

图中按单个 GPU 或 OAM 展示。MI250X 的 128GB 来自两个被 ROCm 独立枚举的 64GB GCD，并不是单个 ROCm 设备的 128GB 本地显存池。四组产品在部分特定 benchmark 中可能接近，也可能差距很大；这张图只负责说明官方竞争关系，不负责给出跨模型、精度和软件的性能结论。

## 都写 128GB，Ryzen AI Max 和 DGX Spark 一样吗

如果真正的问题是模型装不进普通独显，另一条路线是把大容量统一内存放进紧凑设备。

![两种 128GB 统一内存本地 AI 平台](images/05-unified-memory.png)

Ryzen AI Max+ 395 采用 Zen 5 CPU 和 Radeon 8060S，受支持配置可以使用 ROCm，AMD 官方也提供过 Windows + Vulkan 路径。128GB 是系统内存总量，不是 Radeon 8060S 的独占显存；标准 Windows VGM 最多可划出 96GB 作为专用图形内存。

DGX Spark 使用 Arm + Blackwell 的 GB10，官方规格是 128GB LPDDR5x coherent unified system memory，并预装 NVIDIA AI 软件栈。两边都在用大容量共享内存扩大本地模型的可装载范围，但可装下相近规模的模型，不代表首 token、生成速度、批量推理、微调或图像生成性能相同。

## 找到候选型号后，怎样判断能不能用

假设一篇教程写“需要 24GB NVIDIA 显卡”，先不要急着从表里挑一张 AMD 候选。先看这 24GB 是为了装下模型，还是教程依赖某个 CUDA 算子。

我的建议是按这个顺序判断：

1. **先看容量**：模型权重、上下文和运行时空间能不能装下；
2. **再看软件**：目标应用及所需算子，在你的操作系统、GPU 和软件版本上是否有 ROCm、Vulkan 或其他可用实现；
3. **最后看实测和价格**：只比较同模型、同精度、同软件版本和相近配置下的结果。

产品导航负责把搜索范围从十几张卡缩到一两张候选，性能结论仍要由你的真实任务给出。

## 参考资料

- 消费级：[AMD Radeon RX 9000 Series 发布资料](https://www.amd.com/en/newsroom/press-releases/2025-2-28-amd-unveils-next-generation-amd-rdna-4-architectu.html)、[NVIDIA RTX 50 Series 发布资料](https://nvidianews.nvidia.com/news/nvidia-blackwell-geforce-rtx-50-series-opens-new-world-of-ai-computer-graphics)、[RTX 5070 上市资料](https://www.nvidia.com/en-us/geforce/news/rtx-5070-out-now/)、[RTX 5070 Family 规格](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/rtx-5070-family/)、[RX 9060 XT 发布资料](https://www.amd.com/en/newsroom/press-releases/2025-5-20-amd-introduces-new-radeon-graphics-cards-and-ryzen.html)、[RTX 5060 Ti 发布资料](https://nvidianews.nvidia.com/news/nvidia-blackwell-geforce-rtx-arrives-for-every-gamer-starting-at-299)、[RX 7900 XTX 规格](https://www.amd.com/en/products/graphics/desktops/radeon/7000-series/amd-radeon-rx-7900xtx.html)、[RTX 4080 规格](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4080-family/)
- 工作站：[AMD Radeon AI PRO R9700](https://www.amd.com/en/products/graphics/workstations/radeon-ai-pro/ai-9000-series/amd-radeon-ai-pro-r9700.html)、[NVIDIA RTX PRO 4500 Blackwell Workstation Edition](https://www.nvidia.com/en-us/products/workstations/professional-desktop-gpus/rtx-pro-4500/)、[AMD Radeon PRO W7900](https://www.amd.com/en/products/graphics/workstations/radeon-pro/w7900.html)、[NVIDIA RTX 6000 Ada Generation](https://www.nvidia.com/en-us/design-visualization/rtx-6000/)、[本项目 R9700 PyTorch / Transformers 实测](https://github.com/AMD-AIM/zhihu_rednote_articles/tree/main/zhihu/01-huggingface-on-amd)、[本项目 R9700 llama.cpp 记录](https://github.com/AMD-AIM/zhihu_rednote_articles/tree/main/zhihu/03-amd-workload-rocm-stack)、[本项目 W7900 实测](https://github.com/AMD-AIM/zhihu_rednote_articles/tree/main/zhihu/07-qwen3.8-27b-llamacpp)、[多卡并行策略](https://docs.nvidia.com/nemo-framework/user-guide/24.12/nemotoolkit/features/parallelisms.html)
- 数据中心：[AMD MI200](https://www.amd.com/en/products/accelerators/instinct/mi200.html)、[NVIDIA A100](https://www.nvidia.com/en-us/data-center/a100/)、[AMD MI300X](https://www.amd.com/en/products/accelerators/instinct/mi300/mi300x.html)、[NVIDIA H100](https://www.nvidia.com/en-us/data-center/h100/)、[AMD MI325X](https://www.amd.com/en/products/accelerators/instinct/mi300/mi325x.html)、[NVIDIA H200](https://www.nvidia.com/en-us/data-center/h200/)、[AMD MI355X](https://www.amd.com/en/products/accelerators/instinct/mi350/mi355x.html)、[NVIDIA Blackwell](https://www.nvidia.com/en-us/data-center/technologies/blackwell-architecture/)
- 统一内存：[AMD Ryzen AI Max+ 395](https://www.amd.com/en/products/processors/laptop/ryzen/ai-300-series/amd-ryzen-ai-max-plus-395.html)、[AMD Variable Graphics Memory FAQ](https://www.amd.com/en/blogs/2025/faqs-amd-variable-graphics-memory-vram-ai-model-sizes-quantization-mcp-more.html)、[AMD Qwen3.8 Windows + Vulkan 路径](https://www.amd.com/en/blogs/2026/run-qwen-3-8-27b-on-amd-ryzen-ai-max-and-radeon-graphics-cards-day-0.html)、[NVIDIA DGX Spark](https://www.nvidia.com/en-us/products/workstations/dgx-spark/)

#AMD #NVIDIA #显卡 #ROCm #CUDA #本地大模型 #人工智能

你平时在教程里看到最多的是哪张 NVIDIA 卡？自己手里又是哪张 AMD 卡，准备跑什么模型或应用？欢迎把型号和任务留在评论区。
