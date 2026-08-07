# 训练和推理场景下，AMD GPU/APU 与 ROCm 软件栈怎么匹配

<!--
内部交付元数据：
- **平台**：知乎
- **类型**：开发者选型框架（第 3 篇）
- **状态**：draft-r3，事实 / 图片 / 语气审计通过，待用户通读
- **来源**：AMD 产品资料、ROCm 7.14 文档、PyTorch / vLLM / Ollama / llama.cpp 官方资料与本地实测
- **范围**：产品家族与软件层级的工程匹配，不做具体型号排行和跨硬件性能比较
-->

都是跑大模型，但本地跑个对话、线上扛并发、做一轮微调，对软硬件的要求差别很大。光用"训练"和"推理"两个标签来分，粒度太粗——模型多大、什么精度、上下文多长、batch 开多少、要不要跨卡，这些才是先要确定的东西。

### 先把训练和推理拆细一点

"训练"和"推理"各自还能再分。下面四个场景经常互相重叠——本地推理也可能开长上下文和并发，单卡微调可以是 LoRA 也可以是全参数——但每个场景首先要确认的事情不一样：

| 场景 | 首先搞清楚什么 |
|---|---|
| 本地推理 | 模型多大、上下文多长、同时跑几路请求、延迟能接受多少 |
| 高并发服务 | 活跃序列数、KV Cache 占用、batch 大小、吞吐目标、稳定性要求 |
| 单卡微调 | 用什么微调方法、什么精度、微批次多大、上下文多长、训练多少参数 |
| 多卡训练与推理 | 模型总规模、全局 batch、怎么并行，跨卡还是跨节点 |

需求定得越具体，后面硬件和软件就越好排除。

### 第一步：装不装得下

内存容量是最先要过的关，但不能只看模型文件多大。

推理时显存里至少要放三样东西：模型权重、KV Cache、运行时临时空间。
特别是KV Cache：上下文变长、同时活跃的序列变多，KV Cache 就会膨胀，调度压力也跟着上来。

训练比推理吃显存多得多。除了权重本身，还得保存梯度、优化器状态（比如 Adam 的一阶和二阶动量）、反向传播过程中保留的激活，以及算子的临时张量和工作区。
粗略说，全参数 FP16 训练的显存峰值可以到权重本身的三到四倍甚至更高[^1][^2]，具体取决于 batch 大小和上下文长度。

LoRA 倒是把基础权重冻结了，只训练低秩适配器[^3]，所以需要算梯度、存优化器状态的参数量大幅减少。但基础模型本身还在显存里，训练过程中的激活也还在，并不是"模型多大就只占多大"。
QLoRA 进一步把冻结的基础权重量化压缩[^4]，不过计算时的中间精度和状态要看具体实现，不能一概而论。

确认装得下之后，下一步是看瓶颈在哪：是带宽不够、数据喂不快；还是算力不够、矩阵乘算不动。
这两个没有固定先后，得拿目标模型实际跑一下或做 profiling 才能判断。
如果单卡放不下，或者吞吐目标要求多卡，那并行策略、卡间通信量、互联带宽就得一起考虑了。

### 几类 AMD 硬件形态差在哪

AMD 这几类产品不是简单的高低档排列，核心区别在于显存从哪来、有多少、软件支持到什么程度。

**Ryzen AI Max 这类 APU**没有独立显存，CPU 和集成 GPU 共用一块 LPDDR5x 物理内存。那 GPU 到底能用到多少？取决于两件事：一是运行时动态映射过来的部分（GPUVM / GTT），二是 BIOS 里预先划出来的固定区域（carve-out）。所以实际可用量要看系统总内存、BIOS 怎么设的、框架支不支持，三个一起确认。[^5]
**Radeon 消费级独显**有自己的 GDDR6 显存，容量买的时候就定了。但同一代架构的不同型号，ROCm 和框架的支持状态可能不一样，得按完整型号去查。
**Radeon AI PRO** 是工作站级独显，比如 R9700 带 32GB GDDR6[^6]，定位本地 AI 开发和推理。如果一台工作站里插多卡，还要看卡之间的拓扑结构，以及软件层面支持哪种并行方式。
**AMD Instinct** 是数据中心用的[^7]，用 HBM，带宽和容量都比 GDDR 高一个量级，有平台级的互联和扩展能力，训练、推理、HPC 都能覆盖。

总之不能光看架构叫 RDNA 还是 CDNA 就下结论，得看具体型号内存够不够、软件支不支持你要跑的东西[^8]。

### 软件不是一个 ROCm 版本号

硬件选完还只是一半。要真正跑起来，软件这边有好几层要对上：你的卡是什么完整型号、操作系统和内核驱动装的什么版本、ROCm 用户态是哪个版本、上面跑的框架是什么、这个场景还需要哪些额外的库。哪一层没对上都可能出问题。

而且不同任务走的软件路径不一样：

- **训练与通用张量推理**一般走 PyTorch。要对的东西比较多：ROCm 版本、PyTorch 版本、Python 版本，还有一个容易漏的——PyTorch 的编译目标（gfx 架构）得跟你的卡匹配。[^9]
- **LLM 离线批处理和在线服务**可以走 vLLM。vLLM 有自己验证过的 GPU + ROCm + PyTorch + Python 组合，最好直接用它推荐的镜像，自己拼版本容易踩坑。[^10][^11]
- **本地量化模型运行**可以用 Ollama[^12] 或 llama.cpp[^13]，但有个容易搞混的地方：它们实际走的可能是 HIP/ROCm 后端，也可能是 Vulkan 后端。Vulkan 完全不经过 ROCm 用户态，这是两条不同的路径，哪条能用、效果怎么样，得分开确认。
- **多 GPU 训练或推理**由框架选并行策略，底层的集合通信——all-reduce、all-gather、reduce-scatter 这些——走 RCCL[^14]。到了这一步，PCIe 还是 xGMI、要不要跨节点，都变成组合的一部分了。

还有一个容易忽略的地方：用容器跑的时候，镜像里可以把 Python、ROCm 用户态库、框架和依赖都锁住，但宿主机那边的内核驱动、设备权限、GPU 型号和物理互联，容器是管不了的。验证的时候要把宿主和镜像一起看，不是容器能启动就说明没问题。[^15]

### 实际验证怎么做

先用完整设备型号查出 gfx 编译目标，这是后面所有事的起点。然后去[ROCm 兼容矩阵](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)里核对这个型号支持什么 OS、什么驱动、什么 ROCm 版本。接着看你要用的框架——PyTorch、vLLM、Ollama、llama.cpp 还是别的——版本和后端能不能对上。如果用容器，多一步：先确认宿主机已经能把 GPU 暴露出来，再去管镜像里的用户态环境。最后跑一个最小测试，哪怕就一个 tensor 运算或者最小模型的一次推理，确认调用的确实是 GPU 而不是在走 CPU fallback。

笔者在 Ryzen AI Max+ 395 上用 ROCm 7.2.3 和 llama.cpp HIP 后端跑过 Q4 GGUF 量化模型；也在 Radeon AI PRO R9700 上用 ROCm 7.2.0、PyTorch 2.9.1、Transformers 5.14.1，通过 `HIP_VISIBLE_DEVICES` 只暴露一张卡，验证过单卡推理。这两条记录只能说明这两个特定环境跑通了，跟当前 ROCm 7.14 的官方支持状态是两回事，不能互相替代。

说到底，能不能用取决于你的具体设备、软件版本和实际要跑的任务，不是看到 RDNA 或 CDNA 或某个 gfx 代号就能下结论的。

具体操作可以继续参考：完整型号与 gfx 的关系见 [一文讲清 AMD GPU 显卡型号及其代号 gfx](https://zhuanlan.zhihu.com/p/2067663713826612548)；CU、Wavefront、LDS 和独显 / APU 内存路径见 [AMD GPU/APU AI 架构入门](https://zhuanlan.zhihu.com/p/2068399264997315488)；ROCm 与 PyTorch 的安装和最小验证步骤见 [AMD ROCm 与 PyTorch 安装指南](https://zhuanlan.zhihu.com/p/2068740074364260377)。

#AMD #ROCm #GPU #APU #PyTorch #大模型 #人工智能

参考资料：

[^1]: [Hugging Face: Model training anatomy](https://huggingface.co/docs/transformers/model_memory_anatomy)
[^2]: [PyTorch Autograd mechanics](https://docs.pytorch.org/docs/stable/notes/autograd.html)
[^3]: [Hugging Face PEFT: LoRA](https://huggingface.co/docs/peft/main/conceptual_guides/lora)
[^4]: [Hugging Face: bitsandbytes 量化](https://huggingface.co/docs/transformers/main/quantization/bitsandbytes)
[^5]: [ROCm RDNA 3.5 system optimization](https://rocm.docs.amd.com/en/latest/reference/system-optimization/rdna3-5.html)
[^6]: [AMD Radeon AI PRO R9700](https://www.amd.com/en/products/graphics/workstations/radeon-ai-pro/ai-9000-series/amd-radeon-ai-pro-r9700.html)
[^7]: [AMD Instinct accelerators](https://www.amd.com/en/products/accelerators/instinct.html)
[^8]: [ROCm GPU specifications](https://rocm.docs.amd.com/en/docs-7.14.0/reference/gpu-specs.html)
[^9]: [ROCm Compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)
[^10]: [vLLM: Optimization and Tuning](https://docs.vllm.ai/en/latest/configuration/optimization/)
[^11]: [vLLM: GPU installation requirements](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/)
[^12]: [Ollama GPU support](https://docs.ollama.com/gpu)
[^13]: [llama.cpp build backends](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)
[^14]: [ROCm RCCL](https://rocm.docs.amd.com/projects/rccl/en/latest/what-is-rccl.html)
[^15]: [ROCm Docker](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/how-to/docker.html)