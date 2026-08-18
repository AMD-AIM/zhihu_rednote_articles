# 在 AMD Radeon PRO W7900 上跑通 MiniMax H3：ComfyUI 音视频生成实测

<!--
内部状态：
- 阶段：独立事实与读者审阅完成，待用户人工通读
- 实测：单张 W7900，ComfyUI 本地 H3 Q4 Turbo 工作流
- 已确认：864×480、124 帧、24 FPS、5.17 秒、32kHz 双声道音频、生成耗时 276.93 秒
- 待补：merged Q4 Turbo 权重的公开来源、可选的 ComfyUI / GPU 截图、音轨内容人工试听
- 当前使用“运行 / 跑通”，不使用“从零部署”
-->

在 ComfyUI 里按下 `Run` 后，AMD Radeon PRO W7900 用 276.93 秒生成了一段 5.17 秒的视频：画面里，一台红黑色机器人站在计算实验室的工作站前，随着设备灯光亮起转身看向镜头。视频为 864×480、24 FPS，同时带有 32kHz 双声道音频。

这不是调用在线视频生成服务得到的结果。MiniMax H3 的模型、文本编码器和视频、音频 VAE 都加载在本地环境中，画面与声音由同一条 ComfyUI 工作流生成。

MiniMax H3 是一个可以同时处理文本、图像、视频和音频的全模态生成模型。和先生成画面、再单独配音的方式不同，它会在一次生成中同时处理视频与声音。模型开放后，[ComfyUI 已经加入 MiniMax H3 的原生节点](https://github.com/Comfy-Org/ComfyUI/pull/15224)，可以在工作流中完成文本生成音视频、首尾帧生成音视频，以及基于参考图像、视频或音频生成音视频。本次实测在这些原生能力之上使用了社区量化权重、4B 文本编码器和对应的自定义节点。

这次实际跑完以后，我最大的感受是：AMD 现在的 AI 软件栈确实已经很 YES 了。ROCm、PyTorch 和 ComfyUI 串起来以后，在 AMD GPU 上运行一个刚开放的音视频模型，并没有想象中那么难。

下面记录的是模型与工作流已经准备完成后的运行过程和结果，不展开 Linux、ROCm、PyTorch 与 ComfyUI 的从零安装。

### 这次 W7900 运行了什么

本次使用一张 48GB 显存的 AMD Radeon PRO W7900。ComfyUI 版本为 0.32.0，PyTorch 版本为 2.9.1。

这次工作流加载的是社区已经合并 Turbo 加速并量化为 Q4_K_M 的 H3 FL2VA 模型，将采样步数设为 8。运行时直接加载这份合并后的 GGUF 权重，没有再单独加载 Turbo 节点。除了扩散模型，工作流还会加载：

- 4B Qwen3-VL 文本编码器与对应的投影文件；
- MiniMax H3 视频 VAE；
- MiniMax H3 音频 VAE。

这里的 Q4 合并权重和 4B 文本编码器都属于社区优化方案，不是 MiniMax H3 的官方默认配置。官方 ComfyUI 工作流使用 32B 文本编码器；本次环境改用体积更小的 [Qwen3-VL-4B](https://huggingface.co/Comfy-Org/Qwen3-VL/blob/main/text_encoders/qwen3vl_4b_fp8_scaled.safetensors)，再通过实验性的 [ClipProj 投影](https://huggingface.co/NicoLab28/ClipProj-MiniMax-H3)转换成 H3 需要的条件输入。

其中，文本编码器负责理解提示词，扩散模型生成包含画面与声音的中间表示，两个 VAE 再分别还原视频和音频，最后由 ComfyUI 合成为 MP4 文件。

### 打开已准备好的工作流，先改提示词

工作流加载完成后，需要改动的主要内容是提示词、分辨率、视频时长和 seed。这次使用的是纯文本生成音视频，没有上传人物照片或其他参考素材。

提示词描述了一台红黑色机器人走进计算实验室、按下工作站电源并转向镜头，同时要求生成伺服电机、脚步、按钮和风扇启动的声音。为了让短视频中的动作保持集中，整段只使用一个连续镜头，没有加入字幕、Logo 或场景切换。

本次生成参数为：

- 分辨率：864×480；
- 帧数：124 帧；
- 帧率：24 FPS；
- 采样步数：8；
- CFG：1.0；
- 采样器：`res_multistep`；
- seed：`2026081801`。

确认模型、提示词和参数后，点击 `Run` 即可把任务加入队列。ComfyUI 会依次完成模型计算、视频解码、音频解码和 MP4 封装。

### W7900 用 4 分 37 秒生成了什么

这次任务从开始执行到保存视频共用了 276.93 秒。最终文件时长为 5.17 秒、24 FPS，并包含 32kHz 双声道音频。

最终视频中的机器人先背对镜头，随后转身，镜头也逐渐推近。工作站灯光、机器人主体和实验室背景在这段运动中保持了连续：

![MiniMax H3 生成的机器人实验室视频](images/04-h3-robot-preview.gif)

[点击这里查看完整生成视频](amd-w7900-minimax-h3.mp4)。

除了画面，视频还生成了持续的双声道音轨。提示词中的转身和镜头运动表现得比较明显，按下电源按钮的细动作则没有完整呈现。整体上，这条工作流已经在单张 W7900 上完成了从文本输入，到视频、声音和最终 MP4 文件的完整链路。

从文本输入，到模型计算、视频与音频解码，再到最终 MP4 文件，这条链路已经在单张 AMD Radeon PRO W7900 上完整运行。AMD GPU 跑通 MiniMax H3，已经是一件可以实际完成的事。

#AMD #ROCm #MiniMaxH3 #ComfyUI #视频生成 #人工智能

参考资料：

- [MiniMax H3 官方模型](https://huggingface.co/MiniMaxAI/MiniMax-H3)
- [ComfyUI：MiniMax H3 原生节点](https://github.com/Comfy-Org/ComfyUI/pull/15224)
- [ComfyUI 格式的 MiniMax H3 模型与 VAE](https://huggingface.co/Comfy-Org/MiniMax-H3)
