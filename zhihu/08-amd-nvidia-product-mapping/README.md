# AI 教程配置全是 NVIDIA，AMD 该看哪张？从 5070 Ti 到 H100 逐一对应

5070 Ti 够用、4090 更稳、训练上 H100。本地跑模型的硬件建议翻来覆去就是这几张 NVIDIA 的卡，但从来没人告诉 A 卡用户，你的 RX 9070 XT 或者 RX 7900 XTX 该去参考谁。

下面四张表，从 $549 的 RX 9070 一直排到数据中心的 MI355X，NVIDIA 每个级别在 AMD 这边分别是哪张卡，一目了然。

---

![产品线总览](images/01-product-lines.png)

## 消费级显卡，N 卡和 A 卡谁对标谁

这张表列出了消费级 NVIDIA 和 AMD 显卡之间的市场位置对应关系，按年份和价位段配对。

| NVIDIA 这边 | 显存 | AMD 对应 | 显存 | 放一起的理由 |
|---|---:|---|---:|---|
| RTX 5070 Ti | 16 GB GDDR7 | RX 9070 XT | 16 GB GDDR6 | 2025 高端，容量相同，N 卡贵 $150 |
| RTX 5070 | 12 GB GDDR7 | RX 9070 | 16 GB GDDR6 | 同周上市，起售都是 $549 |
| RTX 5060 Ti | 8 / 16 GB GDDR7 | RX 9060 XT | 8 / 16 GB GDDR6 | 2025 主流段，都分 8 GB 和 16 GB 两版 |
| RTX 4080 | 16 GB GDDR6X | RX 7900 XTX | 24 GB GDDR6 | 2022 高端 4K；但 XTX 是 AMD 旗舰，4080 是 NVIDIA 次旗舰 |

![消费级对应一览](images/02-consumer-map.png)

RTX 5070 和 RX 9070 首发价格一样，都是 $549，但 A 卡多了 4 GB 显存（16 GB vs 12 GB）。跑大模型时这 4 GB 有时候决定你能不能多装一个尺寸的模型。

选好 GPU 之后，接下来就是适配工作。大模型推理、图像绘制、训练的工具链绝大多数默认走 CUDA，也就是 NVIDIA 专属。拿 A 卡去跑不是换个型号就完事的，得确认你用的那个具体工具（llama.cpp、ComfyUI、Stable Diffusion、PyTorch……）在 AMD 这边有没有能走的路，ROCm、Vulkan、或者工具自己做的适配。

**卡对上了不代表软件就通了。** 我们之前写过几篇适配相关的文章，可以参考：[一文讲清 AMD GPU 显卡型号及其代号 gfx](https://zhuanlan.zhihu.com/p/2067663713826612548)、[AMD ROCm 与 PyTorch 安装指南](https://zhuanlan.zhihu.com/p/2068740074364260377)、[AMD GPU/APU AI 架构入门](https://zhuanlan.zhihu.com/p/2068399264997315488)。

消费卡应该是兄弟们最容易获取到的 GPU。往上还有工作站卡和数据中心 GPU，如果你关心 32 GB 以上的大容量方案，下面两节覆盖了这部分。

## 工作站卡，RTX PRO 和 Radeon PRO 怎么对应

这张表列出了工作站级别 NVIDIA 和 AMD 显卡的对应关系。

| NVIDIA 这边 | 显存 | AMD 对应 | 显存 | 放一起的理由 |
|---|---:|---|---:|---|
| RTX PRO 4500 (Blackwell) | 32 GB GDDR7 ECC | Radeon AI PRO R9700 | 32 GB GDDR6 | 都是 2025 年专业卡，都瞄着本地 AI + 创作这个交叉点 |
| RTX 6000 Ada | 48 GB GDDR6 ECC | Radeon PRO W7900 | 48 GB GDDR6 ECC | 同代旗舰工作站卡，容量相同，用途重叠（渲染 / 设计 / AI） |

![工作站对应一览](images/03-workstation-map.png)

拿几个热门模型实际跑了一下。

R9700（32 GB）跑 Qwen3.6-35B-A3B 的 Q4 量化版，显存占了大约 20 GB，还有 10 GB 以上的余量给上下文。

W7900（48 GB）跑 Qwen3.8-27B 的 Q4 量化版，只用了约 19 GB 显存，模型所有层都在 GPU 上跑，完全不需要退回 CPU。

以下粗略算一下显存占用。

- **30B 级模型 Q4 量化** ≈ 20 GB 权重。放进 32 GB 卡后还剩 10+ GB 给上下文和运行开销，够用。
- **70B 级模型 Q4 量化** ≈ 40–45 GB 权重。塞进 48 GB 卡后余量只剩几个 GB，上下文稍长就可能爆显存。

---

## 数据中心，AMD Instinct 和 NVIDIA 的官方对标关系

| NVIDIA | 显存 | AMD Instinct | 显存 | 备注 |
|---|---:|---|---:|---|
| A100 | 80 GB HBM2e | MI250X | 2×64 GB HBM2e | 上一代对应 |
| H100 | 80 GB HBM3 | MI300X | 192 GB HBM3 | 当前主力对应 |
| H200 | 141 GB HBM3e | MI325X | 256 GB HBM3e | H100 大容量升级 vs MI300X 大容量升级 |
| B200 | 180 GB HBM3e | MI355X | 288 GB HBM3e | 两边最新一代 |

![数据中心对应一览](images/04-datacenter-map.png)

MI250X 物理上是两个 64 GB 芯片封在一起，系统会把它们当成两张独立 GPU 来用，不是一整块 128 GB。实际部署中分配显存需要按单个 64 GB 来规划。

## 统一内存，同样 128 GB 但架构和软件生态完全不同

独显的显存就那么大，16 GB、24 GB、48 GB 封顶。有没有别的办法？有，让 CPU 和 GPU 共享一整块大内存，不再区分显存和内存。这就是统一内存方案，AMD 和 NVIDIA 各出了一个产品。

![两种 128 GB 统一内存方案](images/05-unified-memory.png)

**AMD 这边，Ryzen AI Max+ 395 系统**

Zen 5 CPU 加集成 Radeon 8060S GPU，两者共享一整块 128 GB LPDDR5x 内存，Windows 下最多可以把 96 GB 分给 GPU 用。软件走 ROCm 或 Vulkan。

**NVIDIA 那边，DGX Spark（GB10 Grace Blackwell）**

Arm CPU 加 Blackwell GPU，同样 128 GB LPDDR5x，CPU 和 GPU 直接共享整块内存不需要手动划分额度。

两个方案解决的是同一个问题，让本地设备装得下 70B 甚至更大的模型，普通独显做不到这一点。但装得下不等于跑得一样快，架构不同、软件生态不同，实际推理速度可能差很多。

---

## 找到 A 卡之后

知道该看谁只是第一步，接下来是判断能不能用。以下是建议顺序。

1. **容量够不够。** 你要跑的模型 + 上下文 + 运行开销，装不装得进这张卡的显存或统一内存。前面的显存占用估算可以直接套用。
2. **软件通不通。** 你具体要用的工具在这张 AMD 卡上有没有能跑的后端，ROCm、Vulkan、或者工具自己做的适配。这是 A 卡最大的不确定性，也是其他几篇文章的重点。
3. **速度行不行。** 同一个模型、同一个量化、同一个软件版本，实测对比才有意义。不同条件下的数字没有可比性。

举个具体的例子，我们在 Radeon PRO W7900 上用 ComfyUI 跑通了 MiniMax-H3 的端到端音视频生成，单卡 276 秒出一段 5 秒带同步音效的视频，完整流程在这里：[手把手教你在 AMD Radeon PRO W7900 上运行 MiniMax H3+ComfyUI 音视频生成](https://zhuanlan.zhihu.com/p/2073092993117073920)。

---

## 看看你的

**你手上是哪张 A 卡？打算拿来跑什么？** 评论区说一下型号和想跑的任务，后面实测篇可以优先覆盖被提到最多的组合。

## 参考资料

- 消费级：[AMD Radeon RX 9000 Series 发布资料](https://www.amd.com/en/newsroom/press-releases/2025-2-28-amd-unveils-next-generation-amd-rdna-4-architectu.html)、[NVIDIA RTX 50 Series 发布资料](https://nvidianews.nvidia.com/news/nvidia-blackwell-geforce-rtx-50-series-opens-new-world-of-ai-computer-graphics)、[RTX 5070 上市资料](https://www.nvidia.com/en-us/geforce/news/rtx-5070-out-now/)、[RTX 5070 Family 规格](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/rtx-5070-family/)、[RX 9060 XT 发布资料](https://www.amd.com/en/newsroom/press-releases/2025-5-20-amd-introduces-new-radeon-graphics-cards-and-ryzen.html)、[RTX 5060 Ti 发布资料](https://nvidianews.nvidia.com/news/nvidia-blackwell-geforce-rtx-arrives-for-every-gamer-starting-at-299)、[RX 7900 XTX 规格](https://www.amd.com/en/products/graphics/desktops/radeon/7000-series/amd-radeon-rx-7900xtx.html)、[RTX 4080 规格](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4080-family/)
- 工作站：[AMD Radeon AI PRO R9700](https://www.amd.com/en/products/graphics/workstations/radeon-ai-pro/ai-9000-series/amd-radeon-ai-pro-r9700.html)、[NVIDIA RTX PRO 4500 Blackwell Workstation Edition](https://www.nvidia.com/en-us/products/workstations/professional-desktop-gpus/rtx-pro-4500/)、[AMD Radeon PRO W7900](https://www.amd.com/en/products/graphics/workstations/radeon-pro/w7900.html)、[NVIDIA RTX 6000 Ada Generation](https://www.nvidia.com/en-us/design-visualization/rtx-6000/)、[本项目 R9700 PyTorch / Transformers 实测](https://github.com/AMD-AIM/zhihu_rednote_articles/tree/main/zhihu/01-huggingface-on-amd)、[本项目 R9700 llama.cpp 记录](https://github.com/AMD-AIM/zhihu_rednote_articles/tree/main/zhihu/03-amd-workload-rocm-stack)、[本项目 W7900 实测](https://github.com/AMD-AIM/zhihu_rednote_articles/tree/main/zhihu/07-qwen3.8-27b-llamacpp)、[多卡并行策略](https://docs.nvidia.com/nemo-framework/user-guide/24.12/nemotoolkit/features/parallelisms.html)
- 数据中心：[AMD MI200](https://www.amd.com/en/products/accelerators/instinct/mi200.html)、[NVIDIA A100](https://www.nvidia.com/en-us/data-center/a100/)、[AMD MI300X](https://www.amd.com/en/products/accelerators/instinct/mi300/mi300x.html)、[NVIDIA H100](https://www.nvidia.com/en-us/data-center/h100/)、[AMD MI325X](https://www.amd.com/en/products/accelerators/instinct/mi300/mi325x.html)、[NVIDIA H200](https://www.nvidia.com/en-us/data-center/h200/)、[AMD MI355X](https://www.amd.com/en/products/accelerators/instinct/mi350/mi355x.html)、[NVIDIA Blackwell](https://www.nvidia.com/en-us/data-center/technologies/blackwell-architecture/)
- 统一内存：[AMD Ryzen AI Max+ 395](https://www.amd.com/en/products/processors/laptop/ryzen/ai-300-series/amd-ryzen-ai-max-plus-395.html)、[AMD Variable Graphics Memory FAQ](https://www.amd.com/en/blogs/2025/faqs-amd-variable-graphics-memory-vram-ai-model-sizes-quantization-mcp-more.html)、[AMD Qwen3.8 Windows + Vulkan 路径](https://www.amd.com/en/blogs/2026/run-qwen-3-8-27b-on-amd-ryzen-ai-max-and-radeon-graphics-cards-day-0.html)、[NVIDIA DGX Spark](https://www.nvidia.com/en-us/products/workstations/dgx-spark/)

#AMD #NVIDIA #显卡 #ROCm #CUDA #本地大模型 #人工智能