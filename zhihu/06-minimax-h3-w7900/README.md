# 在 AMD Radeon PRO W7900 上跑通 MiniMax H3：ComfyUI 音视频生成实测

ComfyUI 是目前最主流的本地 AI 生成工具之一。它把整个生成过程拆成一个个节点——模型加载、提示词输入、采样、解码、保存——节点之间连线就是数据流向。不用写代码，打开工作流就能看到每一步在做什么。

但 ComfyUI 社区长期围绕 NVIDIA 生态运转。教程默认 CUDA，模型优先适配 N 卡，一个新模型要在 AMD GPU 上跑起来，模型格式、节点兼容、显存占用都得重新验证。这不是装个驱动就能解决的事。

MiniMax H3 是最近刚进入 ComfyUI 的一个全模态生成模型。它的特殊之处在于：一次推理同时输出视频和音频，画面和声音在生成阶段就是绑定的，不是先出片再配音。这跟之前那些只管画面、声音另想办法的视频模型是两回事。

那么问题就是：一张 48GB 的 AMD Radeon PRO W7900，能不能把 H3 这条音视频生成链路完整跑下来？

能。这次测试中，W7900 用 276.93 秒生成了一段 5.17 秒的视频：一台红黑色机器人站在计算实验室工作站前，设备灯光亮起，机器人转身看向镜头。输出 864×480、24 FPS，带 32kHz 双声道音频。画面和声音来自同一条本地工作流——所有计算发生在这一张卡上，没有调用任何在线服务。

跑完之后最大的感受是：AMD 的 AI 软件栈现在真的能用了。ROCm、PyTorch、ComfyUI 三层串通之后，在 A 卡上跑一个刚开放的多模态模型，踩的坑比预想中少得多。

接下来的内容从模型和工作流就位之后开始，直接进运行过程和生成结果。环境搭建（Linux、ROCm、PyTorch、ComfyUI 的从零配置）本篇不展开。

### 从文字到视频和声音：H3 的生成链路

把这条工作流想成一条生产线。提示词进去之后，经过三段处理。

第一段，文本编码器。它负责把你写的场景、动作、声音要求翻译成模型能读懂的数学表示。你可以理解为：人话进去，数字出来。

第二段，扩散模型。这是真正"生成"的环节。它从一堆噪声出发，按照文本编码器给的条件，一步步把噪声修成包含画面和声音的中间结果。整条链路里最吃显存、最花时间的就是这一步。

第三段，解码。H3 有两个解码器——一个负责把中间结果还原成视频帧，另一个还原成音频波形。这两个解码器在技术上叫 VAE。最后 ComfyUI 把视频和音频封装成一个 MP4 文件。

三步连起来就是：**理解文字 → 生成画面和声音 → 还原并保存。**

### 怎样把这条链路塞进一张 48GB 的卡

MiniMax H3 开放后，[ComfyUI 已经加入了对应的原生节点](https://github.com/Comfy-Org/ComfyUI/pull/15224)。但官方默认配置需要的显存远超 48GB——光文本编码器就是一个 320 亿参数的模型。要在单张 W7900 上跑通，三段中的每一段都得想办法压。

本次测试使用 ComfyUI 0.32.0、PyTorch 2.9.1，在官方节点基础上换了一套社区提供的轻量配置。下面逐段说明。

**扩散模型：Turbo 加速 + Q4 量化，两刀下去。**

第一刀砍步数。普通扩散模型要走几十步逐渐去噪，而社区训练了一个叫 Turbo 的加速方案（基于 LightX2V），让模型学会用 8 步就接近最终结果。这个加速方案本来是一份单独的 LoRA 权重——LoRA 可以理解为"贴"在主模型上的一小块补丁——但 ChrisColeTech 发布的版本已经提前把补丁融合进了主模型，运行时不需要额外加载。

第二刀砍体积。融合后的模型被量化为 Q4_K_M 精度，保存成 GGUF 格式（一种便于保存和加载量化模型的权重文件格式）。Q4 的意思是用 4-bit 精度存储权重，整个核心模型压缩到约 11.4GB。代价是可能丢失部分生成细节，但换来的是显存占用大幅降低。

**文本编码器：从 32B 换到 4B。**

官方用的是约 320 亿参数的编码器，这里换成约 40 亿参数的 [Qwen3-VL-4B](https://huggingface.co/Comfy-Org/Qwen3-VL/blob/main/text_encoders/qwen3vl_4b_fp8_scaled.safetensors)。参数量砍到八分之一，但输出格式对不上 H3 的输入——所以需要一个实验性的 [ClipProj 投影层](https://huggingface.co/NicoLab28/ClipProj-MiniMax-H3) 做格式转换。

**解码器：不动。**

视频 VAE 和音频 VAE 继续使用 MiniMax 官方发布的权重。这部分本身占用不大，不需要额外压缩。

总结一下：Turbo 解决速度，Q4 解决体积，4B 编码器解决参数量。三个优化叠在一起，才让整条链路在 48GB 显存里跑得动。

本次工作流实际加载的五个文件：

- [H3 FL2VA Turbo Q4_K_M 核心模型](https://huggingface.co/ChrisColeTech/minimax-h3-turbo-GGUF/resolve/main/split/diffusion_models/minimax_h3_fl2va_turbo_Q4_K_M.gguf)
- [Qwen3-VL-4B FP8 文本编码器](https://huggingface.co/ChrisColeTech/minimax-h3-turbo-GGUF/resolve/main/split/text_encoders/qwen3vl_4b_fp8_scaled.safetensors)
- [4B ClipProj 投影模型](https://huggingface.co/NicoLab28/ClipProj-MiniMax-H3/blob/main/mmh3-4b-ClipProj-celeb-mlp.safetensors)
- [MiniMax H3 视频 VAE](https://huggingface.co/ChrisColeTech/minimax-h3-turbo-GGUF/resolve/main/split/vae/minimax_h3_video_vae_fp16.safetensors)
- [MiniMax H3 音频 VAE](https://huggingface.co/ChrisColeTech/minimax-h3-turbo-GGUF/resolve/main/split/vae/minimax_h3_audio_vae_fp32.safetensors)


加载 GGUF 格式的核心模型需要使用更新后的 [ComfyUI-GGUF-Loader](https://github.com/ChrisColeTech/ComfyUI-GGUF-Loader)。ComfyUI 中的节点顺序对应前面的三段链路：加载核心模型 → 4B 编码器 + ClipProj 处理提示词 → 创建 124 帧音视频空间 → 8 步采样 → 视频解码 + 音频解码 → 合成 MP4。

### 提示词：一个五秒的连续镜头

这次使用纯文本生成模式，没有提供参考图片、视频或音频。画面、动作和声音全部由一段提示词驱动。

写提示词时的思路很简单：五秒钟，一个连续镜头，让机器人完成一个完整动作。具体描述是一台红黑色机器人走进计算实验室，按下工作站电源，转身看向镜头。声音部分要求伺服电机、脚步、按钮按压和风扇启动的音效。动作集中、镜头不切，给模型一个明确且有限的任务。

完整提示词：

```text
Photorealistic cinematic video, one continuous shot with no cuts. Inside a clean
futuristic computing laboratory at night, a compact red-and-black service robot
walks toward a glowing workstation, raises one hand and presses the power button.
Cooling fans spin up, soft red light travels through the machine, and the robot
turns toward the camera with a small confident nod. The camera slowly dollies from
a medium-wide shot to a closer view. Stable robot design, realistic brushed metal,
natural reflections, precise mechanical movement, cinematic lighting, no people,
no text, no logo, no watermark. Audio: soft servo movements, metal footsteps, a
clear button click, cooling fans gradually spinning up, and a restrained electronic
ambience. No dialogue.
```

生成参数：

- 分辨率 864×480，124 帧，24 FPS，最终时长 5.17 秒；
- 采样步数 8（对应 Turbo 加速），CFG 1.0，采样器 `res_multistep`；
- seed `2026081801`，固定种子便于复现。

确认无误，点 Run。接下来就是等 W7900 把这条链路跑完。

### 生成结果

从任务开始到 MP4 保存完成，W7900 用了 276.93 秒，也就是约四分半钟。

时间主要花在扩散采样阶段。8 步采样要在 48GB 显存里反复搬运量化后的模型权重，这一段大约占了总时长的九成以上。文本编码和 VAE 解码都很快，各自几秒就结束了。也就是说，如果未来量化方案或显存带宽有进步，这个总时长还有很大的压缩余地。

先看画面。视频开头机器人背对镜头站在工作站前，随后转身，镜头同步推近。实验室灯光、机器人的红黑配色、背景设备细节，在整段运动中保持了连续，没有闪烁或结构崩坏。作为 Q4 量化、8 步采样的结果，画面稳定性比我预期的好。

![MiniMax H3 生成的机器人实验室视频](images/04-h3-robot-preview.gif)

[点击这里查看完整生成视频（含音频）](amd-w7900-minimax-h3.mp4)。

再听声音。能明确听到机械伺服运转的底噪、风扇从静止逐渐加速的声音，以及转身时伴随的机械响动。这些声音在时间上和画面动作基本对得上。不过，提示词里要求的脚步落地和按钮按压没有出现——模型对持续性的环境音表现明显好于短促的离散音效。

再对照提示词看：转身和镜头推进都明确完成了，但"走进实验室"这个前置动作被跳过了，视频开头机器人已经在工作站前站定。可以判断出：H3 似乎会自行取舍哪些动作能在时长内完成——给了太多任务时，它优先保证核心动作的完整，而不是把所有动作压缩到每一帧里。

整体来说，这个结果处在"能看、能听、时间对得上"的水平。五秒、480p、量化模型——不是成品，但它证明链路是通的，出来的东西是有内容的。

### 工作流与模型来源

本次使用的工作流和全部模型权重均可公开获取。提示词换成你自己的场景描述，调整分辨率和时长，点 Run 就能生成。所有计算发生在本地，不需要调用任何在线服务。见参考资料。

### 最后

这次跑通 MiniMax H3 之后，我比较确定一件事：AMD GPU 做本地 AI 生成，已经不是在问"能不能跑"了。ROCm、PyTorch、ComfyUI 串起来以后，一个刚开放的全模态模型可以在单张 W7900 上走完从文字到音视频的完整链路。这件事本身就是答案。

这篇只覆盖了最基础的文本生成音视频。H3 还有首尾帧驱动、参考图像驱动、参考音频驱动——每个新能力都意味着新的显存压力和新的调试。后面有值得写的结果，我会继续发。

如果你手上有 W7900 或者其他 48GB 以上的 AMD 卡，现在就可以试。

#AMD #ROCm #MiniMaxH3 #ComfyUI #视频生成 #人工智能

参考资料：

- [MiniMax H3 官方模型](https://huggingface.co/MiniMaxAI/MiniMax-H3)
- [ComfyUI 格式的 H3 模型与 VAE](https://huggingface.co/Comfy-Org/MiniMax-H3)
- [ComfyUI MiniMax H3 原生节点](https://github.com/Comfy-Org/ComfyUI/pull/15224)
- [ChrisColeTech MiniMax H3 Turbo Q4 模型页](https://huggingface.co/ChrisColeTech/minimax-h3-turbo-GGUF)
