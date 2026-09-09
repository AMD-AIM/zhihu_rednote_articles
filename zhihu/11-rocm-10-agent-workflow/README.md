# ROCm 10 把 Agent 接进开发工作流：哪些能力已发布，哪些仍在预览

大家有没有这样的体会？在AMD上做AI开发，总是比CUDA那边费劲不少。写ROCm相关的代码，API和参数经常对不上，哪怕有一次对上了，考虑到AMD还总在更新，今天能跑的可能过段时间就不行了。自己装环境配ROCm，版本、驱动、框架之间的兼容关系一直是个坑。这也不奇怪，一直以来，公网上AMD的资料就是比CUDA少太多，不管是AI还是人，能查到的好内容都有限，哪怕配置更好的搜索mcp或者搜索skills也没用。

不过，近期AMD确实有了动作。

AMD在8月27号发了ROCm 10（版本号从上代的7.x进行了一个大跳跃），正面应对这个问题，致力于让AI Agent更好的在AMD和ROCm上开发。ROCM 10的核心是一系列名为ROCm.AI的套件，它包含三个模块：AMD Skills、ROCm CLI和Hpyerloom。

本文讲解这三个模块是什么定位，有什么内容，能不能用。

## AMD Skills

开头说的第一个问题，Agent写ROCm的东西老是答不对，怎么办？AMD Skills就负责这个。思路挺直接，与其让自己去各个web上搜，不如AMD官方把验证过的知识和操作方法打包好，直接让Agent能够按需加载。

Skills通常比web上搜到的文档更适合Agent，因为文档只是静态的参数和知识，不含逻辑判断，而Skills则带有逻辑判断和指导，能让Agent更容易操作。用下面要讲的Skill举个例子，serving-llms-on-instinct这个Skill，就会让Agent先查你的GPU型号，再估算模型装不装得进显存，接着选对应的serving配置，然后才启动服务，启动完最后还要跑一次测试。这整套是AMD工程师自己调通以后固化下来的，Agent照着走就不用猜了。

AMD Skills的仓库在[amd/skills](https://github.com/amd/skills)，目前正式目录里有8个Skill，覆盖三个方向，分别是本地方向，服务器方向和性能优化方向。

### 本地方向

本地方向的三个Skill都围绕Lemonade展开。Lemonade是AMD官方开发的本地推理框架，底层是llama.cpp和whisper.cpp，在其上封装为OpenAI兼容格式的API。

这三个Skill分别是：local-ai-use、local-ai-app-integration和lemonade-router-builder。

- **local-ai-use**，作用于Agent，让它理解如何操作Lemonade。
- **local-ai-app-integration**，作用于Lemonade本身的调用和组合，让Agent能够以Lemonade作为子进程的方式，把它嵌入进任何应用里。
- **lemonade-router-builder**，作用于推理请求的分发管理。比如隐私数据走本地模型、普通请求走云端。开发者只需要向Agent描述清楚需求，Agent就能基于Skill生成json配置文件。

### 服务器方向

服务器方向是指让Agent在AMD的服务器硬件上部署LLM推理服务，因此这里的Skill就分两种服务器硬件：一个是AMD的数据中心GPU（Instinct），另一个是AMD的服务器CPU（EPYC）。

目前这个Skills方向只支持vLLM框架，SGlang还在后续计划中，因此这个方向的两个Skills，当前是vLLM在两个不同硬件上的指导。

- **serving-llms-on-instinct**，作用于Instinct GPU上的推理部署。前面举的例子就是这个Skill，包含从硬件检查、显存估算、配置选择到启动验证的内容。
- **serving-llms-on-epyc**，作用于EPYC CPU上的推理部署。内容和Instinct GPU上的基本一致，区别是EPYC上需要配合AMD的推理加速插件zentorch，目前限定EPYC 9000系列。

### 性能优化方向

性能优化方向是指服务已经跑起来之后，让Agent帮你找出慢在哪、怎么提速。三个Skill层层递进，分别是magpie-kernel-evaluator、tracelens-analysis-orchestrator和hyperloom-workload-optimizer。

- **magpie-kernel-evaluator**，作用于GPU kernel级别的性能测量和对比。找出最拖速度的kernel，试不同的实现，不光比速度还比正确性，最后验证改完之后整体确实变快了，而不是只看单个kernel的数字好看。
- **tracelens-analysis-orchestrator**，作用于PyTorch profiler的trace分析。把trace丢进去，它会分析kernel耗时分布、算子融合情况、通信开销，输出一份带优先级的报告。
- **hyperloom-workload-optimizer**，作用于完整的推理优化循环，这个在本文后面的hyperloom部分单独展开。

### 安装与现状

安装就一行，需要Node.js

```bash
npx skills add amd/skills
```

装之前可以先看看最新的Skills有哪些

```bash
npx skills add amd/skills --list
```

官方适配的Agent包括Cursor、Claude Code、Codex和Gemini CLI，也可以手动把Skill目录复制到对应位置安装到其他Agent里。PS：目前现有的Skills步骤主要适配Claude Code，其他Agent可能需要调整部分步骤。

不过上面讲的8个Skill不全能从marketplace装到。Cursor、Claude Code和Codex的marketplace里都只上了其中5个，lemonade-router-builder、serving-llms-on-epyc和magpie-kernel-evaluator需要手动安装。

除了上述正式目录里的8个，翻一下仓库还能看到几个开发中的Skills，比如rocm-doctor和apu-memory-tuner，在github上已经能看到有初步代码了，前者看上去是负责ROCm环境自动诊断，后者则是面向Ryzen APU调试内存配置。此外，还能看到更远的Skills，目前是TODO状态，比如hrr-replay-analysis。

## ROCm CLI

开头说的第二个问题，装环境配ROCm一直是个坑，ROCm CLI就是冲着这个来的。它是一个独立的命令行工具，单个二进制文件，因此不需要提前装Python、Rust或者ROCm。在Linux和Windows上都有预编译版本，可以直接装。使用下列命令：

```bash
curl -fsSL https://raw.githubusercontent.com/ROCm/rocm-cli/main/install.sh | sh
```

装完之后直接输入`rocm`，就能进入交互菜单:

```bash
 rocm.ai  local AI control room
 GPU  —  no live telemetry
 ○ Idle — nothing serving


 What would you like to do?
   ⚙  Set up this system   install / update ROCm
   ◆  Serve a model   run a model on your GPU
   ⚕  Diagnose & fix   check GPU, driver & ROCm
 ▸ ◷  Chat   talk to a local or API model
   ▣  Open full dashboard  →   live instruments & every action
   ⚡  Optimize a model   soon

 ↑↓ move   Enter select   d dashboard   q quit
 ```
也可以用`rocm help`看所有命令。常用的几个：

- `rocm examine`查你的GPU、驱动和ROCm环境
- `rocm install sdk`装ROCm
- `rocm serve`起一个本地模型服务。

### 诊断修复与版本管理

基本命令之外，有两组功能跟开头说的痛点直接相关。分别是`diagnose/fix`和`multi-runtime`。

先说diagnose/fix。

diagnose内置了15类AMD环境下的已知故障检查，比如GPU架构不在PyTorch wheel的编译列表里、amdgpu驱动被blacklist了、框架wheel和系统ROCm版本不匹配、iGPU和独显同时暴露导致运行时崩溃，等Top故障。每种故障有评分机制、证据链和对应的修复方案。

fix那边则是和diagnose对应的15个修复方案。现阶段，有4个能自动执行，比如把用户加到render group、清掉不该设的HSA_OVERRIDE_GFX_VERSION，其余的则是打印操作步骤，让用户来执行。

接着来说multi-runtime。

它专门解决的问题正是多版本管理。开头说过，AMD更新快，今天能跑的过段时间可能就不行了。`rocm runtimes`可以同时保留多个ROCm版本，`activate`切换，`rollback`回上一个。

此外，由于每个ROCm安装都很大，装几次磁盘就满了，所以还有`rocm storage`专门管清理，保护正在用的和回退的版本，清掉其余的旧版本。

还有个有意思的，直接在CLI里输入自然语言也行，比如rocm "is rocm installed?"。简单的问题，它会映射成结构化的检查命令。如果是复杂的问题，需要配一个LLM provider让它理解。

### 实际情况

首先，ROCm CLI目前依然是preview状态，现版本号是v0.1.0-preview.1，GitHub上明确标着Tech Preview，此外，AMD Newsroom原话也说的Technology Preview。

然后我实际跑了下，发现examine能识别到系统环境和GPU，架构正确读出了gfx1201，不过GPU名称只显示通用的AMD Device，没细化到具体型号。diagnose倒是扫出了系统里一条真实存在的blacklist amdgpu配置，说明检查逻辑确实在工作。不过，同一台机器上amdgpu模块是正常加载的，rocminfo也能跑，说明这条诊断还没做到结合运行状态去判断。因此，目前确实是骨架搭好了，但是诊断逻辑和细节还在打磨。

## Hyperloom

开头说的第三个问题，推理服务。一般来说，想进行AI infra，得跑profile找出哪里慢，改一版代码或者调个参数，再跑benchmark看快没快、结果对不对，这样一轮一轮手动搞。Hyperloom就是要把这个过程自动化。它做的不是"帮你分析完给你优化建议"。它是自己改了自己测，测完不对自己回滚。

具体来说，Hyperloom跑一个循环。先测一个基线，作为后续所有比较的锚点，然后通过TraceLens分析trace找瓶颈，接着在多个层面尝试优化：调服务参数和配置、修改框架源码、优化关键GPU kernel。
每一步改完，都重新跑benchmark。有效的改动就叠加起来，无效的就回退或者忽视。
此外，Hyperloom还会在更广泛的测试，例如扫不同并发量、或者输入输出长度，确认不是只在某个测试点上好看。
最后，如果跑完后还有时间预算、又或者收益达标收敛，那么就会就再来一轮。直到最终完成。

自己改了自己测，那改坏了怎么办？Hyperloom每次接受一个改动之前，都会过一道accuracy gate。比如说，如果用vLLM或SGLang跑推理服务，然后用Hyperloom优化。那么可以让它用GSM8K（一个标准的数学推理测试集）跑一次准确率检查，然后设置一个范围（比如最多允许下降5个百分点），如果打破，那么就自动回滚这次改动，不管吞吐提升了多少。
如果是自定义任务，Hyperloom没有内置的准确率标准可以套，此时必须由你自己在benchmark脚本里提供一个quality gate。这里有个坑：不提供的话，Hyperloom不是跳过检查继续跑，而是把所有候选优化都判定为质量不达标，全部拒绝。

有了accuracy gate兜底，Hyperloom能尝试的优化就不只是调参数了。它能直接改框架repo的源码，也能从GitHub PR里找到还没合并的改动拿来试。GPU kernel也在Hyperloom能覆盖的范围内，找出最耗时的kernel重写实现。模型在当前环境下根本跑不起来的话，它还会自己尝试修复，从改配置到打补丁到重新编译，一级一级试。

这些过程中的结果和踩过的坑都会存下来，下次遇到类似任务直接复用。

### 具体使用

用Hyperloom之前先看硬件和软件是否在支持范围内。GPU方面，目前验证过的是三款AMD数据中心GPU：MI300X、MI325X和MI355X。Radeon和Ryzen不在验证矩阵里。框架方面，支持SGLang和vLLM。如果你的工作负载不在这两个框架上，可以通过`--framework custom`把自己的代码和测量脚本接进来让Hyperloom优化。ROCm版本要求7.2.x。
PS：虽然GEAK、Magpie这些子组件各自标了ROCm 10.0.0的支持，但Hyperloom整套流程的验证范围还是ROCm 7.2.x，组件能跑和完整闭环验证过不是一回事。

在目标范围内的话，通过PyPI安装：

```bash
pip install hyperloom-inference-optimizer==1.0.0 --target .
```

此外，Docker和裸机部署也支持。

Hyperloom的优化循环由Agent驱动，Agent需要LLM，所以装完包之后还需要配好LLM API。它通过Claude Code运行，默认走Anthropic协议，兼容的网关都能接，也有走OpenAI协议的Codex路线。运行环境还需要Python 3.10+、Node.js、Claude Code、tmux和jq。

启动方式跟一般命令行工具不太一样。Hyperloom跑在Claude Code里面，你需要写一段提示词告诉它你要优化什么。提示词里需要包含模型路径、框架、GPU型号、TP、并发量、输入输出长度、精度，以及时间预算和优化目标。写完贴进去，Agent自己调CLI启动优化循环并持续监控。官方提供3小时和12小时的演示预设可以参考，其中12小时预设的目标收益从之前的30%提到了50%。

## ROCm 10还有什么

本文只聚焦了ROCm.AI这一块，但ROCm 10是个很大的版本。最底层的变化是构建体系换成了TheRock。此外，Windows和Linux也第一次统一到同一套SDK。除此之外还有不少值得关注的更新，比如vLLM和SGLang的官方验证容器、Ryzen AI MAX上的本地微调支持、Roofline分析第一次覆盖到RDNA3、通信库RCCL也拿到了这个版本里最大的单项投入。这些都不在本文范围内，感兴趣的可以看[官方技术博客](https://rocm.blogs.amd.com/ecosystems-and-partners/rocm-x-blog/README.html)。

最后一个亮点：ROCm.AI这三个模块全部是今年才开源的。AMD Skills仓库今年建的，ROCm CLI 8月底才发了第一个preview release，Hyperloom 7月还是tech preview、8月底刚出v1.0.0。三个repo的代码都在GitHub上，MIT协议。你现在去翻这些仓库，看到的就是它们最早期的样子。但反过来说，现在也是最容易参与和反馈的时候。

你在AMD上做AI开发，最想让Agent帮你干什么？装环境？排障？部署模型？还是优化性能？

## 参考资料

- [ROCm 10官方技术博客](https://rocm.blogs.amd.com/ecosystems-and-partners/rocm-x-blog/README.html)
- [AMD ROCm 10 Newsroom](https://newsroom.amd.com/news/rocm-10-software-ai-native-developer-experiences/)
- [AMD Skills](https://github.com/amd/skills)
- [ROCm CLI](https://github.com/ROCm/rocm-cli)
- [ROCm CLI v0.1.0-preview.1](https://github.com/ROCm/rocm-cli/releases/tag/v0.1.0-preview.1)
- [Hyperloom](https://github.com/AMD-AGI/Hyperloom)
- [Hyperloom v1.0.0](https://github.com/AMD-AGI/Hyperloom/releases/tag/v1.0.0)
- [Hyperloom兼容矩阵](https://rocm.docs.amd.com/projects/hyperloom/en/latest/compatibility.html)
