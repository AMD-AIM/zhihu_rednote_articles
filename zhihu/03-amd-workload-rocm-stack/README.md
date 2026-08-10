# 训练和推理场景下，AMD GPU/APU 与 ROCm 软件栈怎么匹配

<!--
内部交付元数据：
- **平台**：知乎
- **类型**：开发者选型框架（第 3 篇）
- **状态**：review-r2，13 条口头反馈与小红书引用要求已落实，待用户通读
- **来源**：AMD 产品资料、ROCm 7.14 文档、PyTorch / vLLM / Ollama / llama.cpp 官方资料与本地实测
- **范围**：产品家族与软件层级的工程匹配，不做具体型号排行和跨硬件性能比较
-->

很多人都在用 AMD GPU 跑大模型，但本地跑个对话、线上扛并发、给模型做微调，这类不同场景，哪怕用的是同一个模型，对显存和软件环境的要求也会差很多。

所以不管做训练还是推理，都要先明确模型多大、使用什么精度、上下文多长、batch 开多少、是否需要跨卡，这类问题，然后去看硬件和软件。

### 具体应用场景细分

训练和推理各自还能再分。下面四个场景经常互相重叠，但每个场景首先要确认的事情不一样：

| 场景 | 首先搞清楚什么 |
|---|---|
| 本地推理 | 模型多大、上下文多长、同时跑几路请求、延迟能接受多少 |
| 高并发服务 | 活跃序列数、KV Cache 占用、batch 大小、吞吐目标、稳定性要求 |
| 单卡微调 | 用什么微调方法、什么精度、多大的 micro batch size 、上下文多长、训练多少参数 |
| 多卡训练与推理 | 模型总规模、全局 batch、怎么并行，跨卡还是跨节点 |

从上表可以看到，本地推理先看模型和 KV Cache，高并发服务还要看同时处理多少请求，微调要把梯度、优化器状态和激活算进去；进入多卡以后，通信和互联也会成为新的限制。需求定得越具体，后面硬件和软件就越好判断。

### 不同任务需要多少显存

内存容量是最先要过的关，但不能只看模型文件多大。

推理时显存里至少要放三样东西：模型权重、KV Cache、运行时临时空间（比如算子的 workspace）。
特别是KV Cache：上下文变长、同时活跃的序列变多，KV Cache 就会膨胀，调度压力也跟着上来。

训练比推理吃显存多得多。除了权重本身，还得保存梯度、优化器状态（比如 Adam 的一阶和二阶动量）、反向传播过程中保留的激活，以及算子的临时张量和 workspace。
全参数训练的显存占用通常远高于模型权重本身[^1][^2]，具体取决于权重、梯度和优化器状态的精度，以及 batch 大小、上下文长度和激活保存策略。

LoRA 倒是把基础权重冻结了，只训练低秩适配器[^3]，所以需要算梯度、存优化器状态的参数量大幅减少。但基础模型本身还在显存里，训练过程中的激活也还在，并不是"模型多大就只占多大"。
QLoRA 进一步把冻结的基础权重量化压缩[^4]，不过计算时的中间精度和状态要看具体实现，不能一概而论。

推理时可以先估算权重文件。例如 32B 模型按 4bit 计算，原始权重约为：

```text
32B × 4bit ÷ 8 ≈ 16GB
```

再加上量化 scale、元数据和格式开销，常见 Q4 GGUF 文件约 20GB；真正运行时还要继续给 KV Cache 和 workspace 留空间。

| 模型规模 | 典型 Q4 GGUF 权重文件 | 具体容量实例 |
|---|---:|---|
| 7B | 约 5GB | 16GB Radeon RX 9070 XT 可以留出较多运行空间[^8] |
| 14B | 约 9GB | 16GB 独显能容纳权重，实际余量取决于上下文和后端 |
| 32B | 约 20GB | 32GB Radeon AI PRO R9700 可继续容纳 KV Cache 和 workspace[^6] |
| 70B | 约 40–45GB | 48GB Radeon PRO W7900 余量已经较紧；128GB 统一内存机器的容量空间更大[^5][^8] |

训练的差距更大。LoRA 和 QLoRA 会减少需要保存梯度和优化器状态的参数范围，但实际占用会随模型、rank、序列长度、micro batch size、精度和训练框架变化。

[AMD显卡到底能不能跑大模型？](https://www.xiaohongshu.com/explore/6a7599e6000000002202c0c2?xsec_token=ABKJZCTAQBzCqo5vv3wr5GYi7kzhgTRDaQoA3CIrmEerk=&xsec_source=pc_search)用 8B 模型举例，给出的量级是 QLoRA 约 12GB、LoRA 约 28GB、全参数微调约 85GB，直观展示了三种方法的差距。显存组成和计算方法还可以继续参考[大模型训练和推理中的显存占用来源](https://zhuanlan.zhihu.com/p/2058222916676989074)、[预估模型训练和推理时的显存](https://zhuanlan.zhihu.com/p/2012270379821979384)和[深度学习模型推理过程所需的显存大小应该如何计算](https://www.zhihu.com/question/453677760/answer/76215193202)。

### 具体 AMD GPU/APU 的内存与应用场景

AMD 的几类产品不是简单的高低档排列，核心区别在于内存从哪来、有多少，以及软件支持到什么程度。

| 产品实例 | 内存形态 | 可以用来理解什么场景 |
|---|---|---|
| Ryzen AI Max+ 395 / Radeon 8060S | 最高 128GB 统一 LPDDR5x | 大容量本地量化推理；GPU 实际可用量还受系统、BIOS 和框架影响[^5][^15] |
| Radeon RX 9070 XT | 16GB GDDR6 | 7B、14B 量化模型，以及容量范围内的 PyTorch 推理[^8] |
| Radeon RX 7900 XTX | 24GB GDDR6 | 更大模型的单卡推理，或根据具体配置开展参数高效微调[^8] |
| Radeon AI PRO R9700 | 32GB GDDR6 | 本地 AI 开发、推理和工作站多卡场景[^6] |
| Radeon PRO W7900 | 48GB GDDR6 | 需要更大单卡本地显存的工作站场景[^8] |
| Instinct MI300X / MI325X / MI355X | 192GB / 256GB / 288GB HBM | 数据中心训练、推理和多 GPU 扩展[^16][^17][^18] |

从这些实例可以看到，同样是 RDNA 架构，16GB、24GB、32GB 和 48GB 对应的容量范围已经不同；进入 Instinct 后，内存形态和平台互联也一起改变。判断时要同时看具体型号、显存和目标软件，不能只看 RDNA 或 CDNA 名称。

### 推理、微调和训练分别用什么软件

硬件容量明确以后，软件可以直接按任务来分：

| 使用场景 | 软件组合 | 直接从哪里开始 |
|---|---|---|
| 本地量化推理 | Ollama、LM Studio 或 llama.cpp | 加载 GGUF 模型后查看实际 GPU 后端；Ollama 可用 `ollama ps` 查看 offload 比例[^12][^13][^19] |
| PyTorch 推理与微调 | Linux + ROCm + PyTorch，微调时再接 PEFT、TRL 等训练库 | 先查[gfx 编译目标](https://zhuanlan.zhihu.com/p/2067663713826612548)，按[安装指南](https://zhuanlan.zhihu.com/p/2068740074364260377)安装，再跑一个 GPU Tensor[^9] |
| LLM 离线批处理和在线服务 | Linux + ROCm + vLLM | 从 AMD ROCm 7.14 的固定验证镜像开始，再执行 `vllm serve <模型>` 启动 OpenAI 兼容 API[^10][^11][^20] |
| 多 GPU 训练或推理 | PyTorch DDP / FSDP 等分布式策略 | 先用 `torchrun --nproc-per-node=<GPU 数量> train.py` 启动；PyTorch 参数写 `nccl`，ROCm 环境底层使用 RCCL[^14] |

从这张表可以看到，本地推理、PyTorch 微调、vLLM 服务和多 GPU 任务有不同的起点，不需要先把所有软件都装一遍。

### 实际验证怎么做

PyTorch 场景可以先查设备，再跑一个最小 Tensor验证。注意 ROCm 版 PyTorch 复用的是 torch.cuda 这套接口，写法和 NVIDIA 环境一样：

```bash
rocminfo | grep -E 'Marketing Name|Name:.*gfx'
python - <<'PY'
import torch

print("available:", torch.cuda.is_available())
print("HIP:", torch.version.hip)
print("device:", torch.cuda.get_device_name(0))

x = torch.randn((1024, 1024), device="cuda")
print("result:", (x @ x).device)
PY
```

ROCm 版 PyTorch 沿用 `torch.cuda` 接口。`available` 为 `True`、HIP 版本非空、结果位于 `cuda:0`，说明这条最小计算链路已经调用 GPU。完整安装和排查过程见 [AMD ROCm 与 PyTorch 安装指南](https://zhuanlan.zhihu.com/p/2068740074364260377)。

如果你不需要 PyTorch 环境、只想快速跑个模型对话，Ollama 这类工具上手更快。可以参考 AMD Ryzen AI Max 的实践步骤[^15]：

```bash
ollama pull qwen3.5:35b
ollama run qwen3.5:35b
ollama ps
```

`ollama ps` 会显示模型使用 CPU 还是 GPU，以及 GPU offload 比例。该文章还给出了统一内存配置和不同模型的本地运行结果。

笔者在 Ryzen AI Max+ 395 上用 ROCm 7.2.3 和 llama.cpp HIP 后端跑过 Q4 GGUF 量化模型；也在 Radeon AI PRO R9700 上用 ROCm 7.2.0、PyTorch 2.9.1、Transformers 5.14.1 验证过单卡推理。两套本地实测的完整跑通过程后续再单独展开。

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
[^15]: [AI Inference on AMD Ryzen AI Max Processor](https://rocm.blogs.amd.com/artificial-intelligence/ryzen-uma-llm/README.html)
[^16]: [AMD Instinct MI300X](https://www.amd.com/en/products/accelerators/instinct/mi300/mi300x.html)
[^17]: [AMD Instinct MI325X](https://www.amd.com/en/products/accelerators/instinct/mi300/mi325x.html)
[^18]: [AMD Instinct MI355X](https://www.amd.com/en/products/accelerators/instinct/mi350/mi355x.html)
[^19]: [LM Studio 0.3.9：ROCm 与 Vulkan 引擎](https://lmstudio.ai/blog/lmstudio-v0.3.9)
[^20]: [ROCm：vLLM inference](https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/inference/vllm.html)