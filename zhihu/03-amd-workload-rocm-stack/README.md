# 训练和推理场景下，AMD GPU/APU 与 ROCm 软件栈怎么匹配

<!--
内部交付元数据：
- **平台**：知乎
- **类型**：开发者选型框架（第 3 篇）
- **状态**：draft-r3，事实 / 图片 / 语气审计通过，待用户通读
- **来源**：AMD 产品资料、ROCm 7.14 文档、PyTorch / vLLM / Ollama / llama.cpp 官方资料与本地实测
- **范围**：产品家族与软件层级的工程匹配，不做具体型号排行和跨硬件性能比较
-->

同样是跑一个大模型，本地单人对话、在线并发服务和微调，对软硬件的要求并不相同。只按训练或推理两个标签分类，容易漏掉更先决定组合的变量：模型规模、精度、上下文、batch、并发，以及是否需要跨卡。

判断时可以沿着一条路径往下走：

```text
工作负载
→ 资源约束
→ 硬件形态
→ 软件层级
→ 兼容性与运行验证
```

### 先把训练和推理继续拆开

下表中的四个代表场景可能互相重叠，但首先要问的问题不同。

| 代表场景 | 首先确认什么 |
|---|---|
| 本地推理 | 模型权重、上下文、同时运行的请求数和延迟目标 |
| 高并发服务 | 活跃序列、KV Cache、batch、吞吐和服务稳定性 |
| 单卡微调 | 微调方法、精度、微批次、上下文和需要训练的参数范围 |
| 多卡训练与推理 | 模型规模、全局 batch、并行方式，以及是否跨卡或跨节点 |

本地推理也可能使用长上下文和并发请求；单卡微调既可能是 LoRA，也可能是全参数训练。任务画像越具体，后面的硬件与软件范围越容易收窄。

### 先确认任务能不能放下

内存容量是第一个可行性门槛，但不能只看模型文件大小。

推理时至少要容纳模型权重、KV Cache 和运行时临时空间。上下文变长、同时活跃的序列增加，KV Cache 和调度压力也会增加。

训练还要保存反向传播需要的状态：

- 模型权重；
- 可训练参数的梯度；
- 优化器状态；
- 为反向传播保留的激活；
- 算子临时张量和工作区。

典型 LoRA 配置会冻结基础权重，主要训练低秩适配器，因此减少梯度和优化器状态的覆盖范围；实际仍以训练库报告的 trainable parameters 为准，基础模型和训练激活也仍然存在。QLoRA 进一步压缩冻结基础模型的存储，但计算精度和中间状态需要按具体实现分别确认。

容量通过以后，再用目标模型的实际运行或 profiling 判断当前阶段更受内存带宽还是矩阵计算限制，两者没有固定先后。只要模型或吞吐目标需要跨卡，并行方式、通信量和互联就必须同时进入判断。

### 几类 AMD 硬件形态差在哪

AMD 的几类硬件首先按内存和部署形态区分，不按一条简单的快慢顺序排列。

- **Ryzen AI Max 这类 APU**：CPU 与集成 GPU 共享 LPDDR5x 物理内存。Linux 应用通过 GPUVM / GTT 动态映射共享内存，BIOS carve-out 则是另行固定预留的区域。需要核对系统实际内存、GPU 可用口径和具体框架支持。
- **Radeon 消费级独显**：GPU 使用独立 GDDR6，本地显存容量固定。需要按完整型号核对当前 ROCm、OS 和框架支持。
- **Radeon AI PRO**：工作站级独显，例如 Radeon AI PRO R9700 配有 32GB GDDR6，面向本地 AI 开发和推理。进入多卡工作站后，还要核对拓扑和软件并行方式。
- **AMD Instinct**：数据中心 GPU 使用 HBM，并提供平台互联和扩展能力，同时覆盖训练、推理和 HPC。

统一内存 APU 的容量来自共享系统内存，独显和 Instinct 则使用本地 GDDR 或 HBM。内存形态、软件支持和工作负载共同决定候选范围，架构名称本身不对应固定用途。

### 软件不是一个 ROCm 版本号

一个可运行组合至少包含：

```text
完整设备型号
→ OS / 内核驱动
→ ROCm 用户态
→ 框架或运行时
→ 场景需要的库
```

不同任务会从不同入口进入：

- **训练与通用张量推理**通常从 PyTorch 进入，需要核对 ROCm、PyTorch、Python 和目标 gfx（编译目标）的组合。
- **LLM 离线批处理和在线服务**可以从 vLLM 进入，需要使用当前验证的 GPU、ROCm、PyTorch、Python 和镜像组合。
- **本地量化模型运行**可以使用 Ollama 或 llama.cpp，并分别确认实际采用 HIP / ROCm 还是 Vulkan 后端。Vulkan 不经过 ROCm 用户态，两条后端路径需要分开核对。
- **多 GPU 训练或推理**由框架选择并行策略，RCCL 执行 all-reduce、all-gather、reduce-scatter 等 collective；PCIe / xGMI 和多节点网络也属于组合的一部分。

容器可以固定 Python、ROCm 用户态库、框架和应用依赖；宿主机仍负责内核驱动、设备权限、GPU 型号和物理互联。验证组合时要同时查看宿主与镜像，而不是只看容器是否启动。

### 四个代表场景怎样走完这条路径

- **本地大模型推理**：根据模型权重、上下文和 KV Cache 得到容量范围，再匹配统一内存 APU 或独立显存 GPU，以及 Ollama、llama.cpp、PyTorch 等实际入口；最后用目标模型的最小链路验证。
- **单卡微调**：根据典型 LoRA、QLoRA 或全参数微调得到权重、激活、梯度和优化器状态范围，再匹配单设备内存形态、PyTorch 与训练库；最后用最小 batch 完成一次 forward / backward。
- **高并发服务**：根据活跃序列、KV Cache、batch 和延迟目标得到容量与吞吐约束，再匹配当前 vLLM 验证的 GPU、ROCm、PyTorch、Python 和镜像版本；最后用目标并发做压力验证。
- **多卡训练与推理**：根据数据并行、参数分片或张量并行得到状态分布和通信量，再匹配服务器或已验证的工作站拓扑、框架策略与 RCCL；最后验证 collective、互联和目标工作负载。

我在 Ryzen AI Max+ 395 的 ROCm 7.2.3、llama.cpp HIP 环境中验证过 Q4 GGUF 量化模型运行；也在 Radeon AI PRO R9700 的 ROCm 7.2.0、PyTorch 2.9.1、Transformers 5.14.1 环境中通过 `HIP_VISIBLE_DEVICES` 只暴露一张 GPU，验证过单卡推理链路。这两条记录只对应各自环境，与当前 ROCm 7.14 的官方支持核对是不同证据。

### 最后把组合逐层锁定

确定候选范围后，按下面的顺序核对：

1. 用完整设备型号确认 gfx（编译目标），再进入具体支持查询。
2. 在当前 [ROCm 兼容矩阵](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html) 中核对型号、OS、驱动与 ROCm。
3. 再核对 PyTorch、vLLM、Ollama、llama.cpp 或训练库的版本和后端。
4. 容器场景先确认宿主机已经能暴露 GPU，再固定用户态环境。
5. 运行最小 Tensor 或目标框架的最小模型链路，确认实际调用 GPU。

最终能否成立，取决于具体设备、软件版本和目标工作负载，而不是 RDNA、CDNA 或一个 gfx 代号单独给出的答案。

具体操作可以继续参考：完整型号与 gfx 的关系见 [一文讲清 AMD GPU 显卡型号及其代号 gfx](https://zhuanlan.zhihu.com/p/2067663713826612548)；CU、Wavefront、LDS 和独显 / APU 内存路径见 [AMD GPU/APU AI 架构入门](https://zhuanlan.zhihu.com/p/2068399264997315488)；ROCm 与 PyTorch 的安装和最小验证步骤见 [AMD ROCm 与 PyTorch 安装指南](https://github.com/AMD-AIM/zhihu_rednote_articles/blob/main/zhihu/AMD%20ROCm%E4%B8%8EPyTorch%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97-%E7%9F%A5%E4%B9%8E%E7%89%88.md)。

#AMD #ROCm #GPU #APU #PyTorch #大模型 #人工智能

参考资料：

- [ROCm Compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)
- [ROCm GPU specifications](https://rocm.docs.amd.com/en/docs-7.14.0/reference/gpu-specs.html)
- [PyTorch Autograd mechanics](https://docs.pytorch.org/docs/stable/notes/autograd.html)
- [Hugging Face：Model training anatomy](https://huggingface.co/docs/transformers/model_memory_anatomy)
- [Hugging Face PEFT：LoRA](https://huggingface.co/docs/peft/main/conceptual_guides/lora)
- [Hugging Face：bitsandbytes 量化](https://huggingface.co/docs/transformers/main/quantization/bitsandbytes)
- [vLLM：Optimization and Tuning](https://docs.vllm.ai/en/latest/configuration/optimization/)
- [vLLM：GPU installation requirements](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/)
- [Ollama GPU support](https://docs.ollama.com/gpu)
- [llama.cpp build backends](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)
- [ROCm RCCL](https://rocm.docs.amd.com/projects/rccl/en/latest/what-is-rccl.html)
- [ROCm Docker](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/how-to/docker.html)
- [ROCm RDNA 3.5 system optimization](https://rocm.docs.amd.com/en/latest/reference/system-optimization/rdna3-5.html)
- [AMD Radeon AI PRO R9700](https://www.amd.com/en/products/graphics/workstations/radeon-ai-pro/ai-9000-series/amd-radeon-ai-pro-r9700.html)
- [AMD Instinct accelerators](https://www.amd.com/en/products/accelerators/instinct.html)
