# 320B 的 GLM-5.3-Flash，我用四张 R9700 跑通了文本和图片

智谱 8 月 26 号发布了 GLM-5.3-Flash，GLM-5 系列第一个原生多模态模型，320B 参数，MoE 架构每个 token 只激活 18B，文本、图片、视频都能处理，权重以 MIT 协议开放。发布前它以 Ox Alpha 的代号在 OpenRouter 上匿名测试了一段时间，公开之后大家才知道之前用的是一个 320B 的模型。

权重开放了，但软件端还在早期。llama.cpp 主线不支持 GLM-5.3-Flash，要跑起来得切到 GLM5Next 分支，还没合并进主线。Unsloth 发布了 UD-Q2_K_XL 量化权重，320B 压到四个分片合计约 108.7GB，加上 1.13GB 的 F16 视觉投影器，四张 32GB 显存的卡理论上能装下。

我手上有四张 Radeon AI PRO R9700，正好 32GB × 4。用 Vulkan 后端加 layer split 把模型分到四张卡上，文本端跑通了短回答和 7266-token 的长文检索，图片理解也能返回正确内容。GLM-5.3-Flash 的托管服务已经支持视频输入，但本地走 GGUF 量化这条路，GLM5Next 分支还没接通原生视频处理，这次验证覆盖文本和静态图片。

## 环境和前置检查

四张 R9700 物理显存合计约 119.4GiB，权重和投影器加起来约 110GB，账面上有余量。但推理时 KV cache 和中间结果还要额外占空间，实际并不宽裕。

**测试环境**

- GPU 四张 Radeon AI PRO R9700，每张 32GB
- 系统内存约 502GiB
- 模型和投影器放在独立数据盘
- 推理后端 Vulkan RADV
- 模型用 layer split 分到四张卡
- CPU 线程 32
- Flash Attention 和 MTP 均关闭

下载前先确认磁盘空间，加载前检查显存和 GPU 进程。

    df -h ./models
    rocm-smi --showmeminfo vram --showpidgpus

模型目录至少留 120GB。rocm-smi 输出里，计划使用的四张卡显存占用接近 0、没有其他进程占用 DRM 设备，就可以继续。偶尔会看到使用 0 个 DRM 设备的瞬时 KFD 条目，不影响，看实际显存占用就行。

## 固定权重和源码版本

GLM5Next 分支和模型仓库都还在更新，不锁版本的话，同样的步骤隔几天结果可能不一样。先固定模型 revision，再固定 llama.cpp commit。

模型用 Unsloth 发布的 GLM-5.3-Flash-GGUF 仓库，revision 锁定在 2975ab414d30340466d8c51533c6e91f0cca64c1，量化选 UD-Q2_K_XL，投影器用 mmproj-F16.gguf。权重和投影器分两条命令下载。

    mkdir -p ./models/GLM-5.3-Flash-GGUF

    hf download unsloth/GLM-5.3-Flash-GGUF \
      --revision 2975ab414d30340466d8c51533c6e91f0cca64c1 \
      --include 'UD-Q2_K_XL/*' \
      --local-dir ./models/GLM-5.3-Flash-GGUF \
      --max-workers 2

    hf download unsloth/GLM-5.3-Flash-GGUF \
      mmproj-F16.gguf \
      --revision 2975ab414d30340466d8c51533c6e91f0cca64c1 \
      --local-dir ./models/GLM-5.3-Flash-GGUF

下载完成后目录里应该有四个编号分片和一个 mmproj-F16.gguf。

源码用 Unsloth 的 glm5next/upstream 分支，锁定到以下 commit。

    git clone --single-branch \
      --branch glm5next/upstream \
      https://github.com/unslothai/llama.cpp.git

    cd llama.cpp
    git checkout --detach 949f7efb097eb20ef36fecdb1afaebff9a4ae7ed
    git rev-parse HEAD

最后一条命令输出的 hash 和上面一致就对了。这个分支随时可能合并或更新，文末附了 PR 链接。

## 构建 Vulkan 版本

构建在 Ubuntu 24.04 容器里完成，构建目录挂回宿主机。容器内安装依赖需要 root，挂载目录只指向源码目录就行。


    apt-get update
    apt-get install -y \
      cmake build-essential ccache libvulkan-dev vulkan-tools \
      glslc glslang-tools libshaderc-dev spirv-tools spirv-headers

    cmake -S . -B build-vulkan \
      -DGGML_VULKAN=ON \
      -DGGML_NATIVE=OFF \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLAMA_CURL=OFF

    cmake --build build-vulkan \
      --config Release \
      -j32 \
      --target llama-cli llama-server llama-mtmd-cli

构建完成后 build-vulkan/bin 下会出现 llama-cli、llama-server 和 llama-mtmd-cli 三个二进制。

这些二进制依赖同目录下的共享库，直接启动可能报找不到 libllama-cli-impl.so，先设动态库路径。

    export LD_LIBRARY_PATH=$PWD/build-vulkan/bin${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
    ./build-vulkan/bin/llama-cli --list-devices

设备列表里能看到四张 AMD Radeon AI PRO R9700，Vulkan0 到 Vulkan3，后面加载模型按这个顺序指定。

## 跑一个最小文本任务

先用短上下文、单轮提示跑一次，确认模型能加载、正确分到四张卡、生成最终回答。

    ./build-vulkan/bin/llama-cli \
      -m ../models/GLM-5.3-Flash-GGUF/UD-Q2_K_XL/GLM-5.3-Flash-UD-Q2_K_XL-00001-of-00004.gguf \
      -dev Vulkan0,Vulkan1,Vulkan2,Vulkan3 \
      -sm layer \
      -ngl auto \
      -fit on \
      -fitt 2048 \
      -c 4096 \
      -t 32 \
      -tb 32 \
      -fa off \
      -n 512 \
      --temp 0 \
      --seed 0 \
      --reasoning-effort low \
      --reasoning-budget 256 \
      --single-turn \
      --no-mmproj \
      -p '请只用一句中文回答，本地 GLM-5.3-Flash 文本推理已经运行。'

模型返回了一句

    本地GLM-5.3-Flash文本推理已成功运行。

文本推理通了，下一步测长上下文。

## 长上下文检索

短回答只覆盖基础加载和解码。我生成了 240 条结构相近的记录，验证码分别埋在第 17 条和第 233 条，要求模型只返回这两项。输入 tokenize 后共 7266 tokens。

模型返回

    017=LANTERN-17017；233=ORBIT-99233

完全一致，truncated 为 0，没有重复 token 或乱码。

## 静态图片验证

文本通过后加入视觉投影器 mmproj-F16.gguf。测试图是一张合成探针，顶部写着 HELLO 42，左侧蓝色正方形，右侧红色圆形。

![本地视觉探针](images/vision-probe.png)

用 llama-mtmd-cli 加载，视觉投影器放在 Vulkan0。

    ./build-vulkan/bin/llama-mtmd-cli \
      -m ../models/GLM-5.3-Flash-GGUF/UD-Q2_K_XL/GLM-5.3-Flash-UD-Q2_K_XL-00001-of-00004.gguf \
      -mm ../models/GLM-5.3-Flash-GGUF/mmproj-F16.gguf \
      --image ./vision-probe.png \
      -mmdev Vulkan0 \
      -dev Vulkan0,Vulkan1,Vulkan2,Vulkan3 \
      -sm layer \
      -ngl auto \
      -fit on \
      -fitt 2048 \
      -c 8192 \
      -t 32 \
      -tb 32 \
      -fa off \
      -n 512 \
      --temp 0 \
      --seed 0 \
      --jinja \
      -p '请读取图片，只用一句中文准确回答顶部文字、左侧形状及颜色、右侧形状及颜色。'

模型返回

    顶部文字为HELLO 42，左侧是一个蓝色正方形，右侧是一个红色圆形。

三项全对。当前 Vulkan 后端不支持视觉模块里的部分 F32 算子，这些计算回退到了 CPU，功能正常，但不是纯 GPU 推理。

## 换一张真实视频帧再测

合成探针结构简单，再用一张真实画面试试。我们 8 月 18 日在知乎发过一篇[《手把手教你在 AMD Radeon PRO W7900 上丝滑运行 MiniMax H3+ComfyUI 音视频生成》](https://zhuanlan.zhihu.com/p/2073092993117073920)，成片里是一台红黑机器人站在夜间实验室的工作站前。我拿了那段视频的第 36 帧作为静态图片输入。

![MiniMax H3 视频第 36 帧](images/minimax-h3-frame-036.png)

模型识别出机器人有一只手臂伸向中央面板，判断中央圆形部件已经发出红光，两部分和连续帧吻合。但它同时写了手指已经触及中央控制面板。逐帧放大手臂区域后，只能确认手臂从低位抬起并靠近面板，画面里没有清晰的按钮轮廓，也没有足够的遮挡、形变或位移能证明接触。

模型发现了人工复核时漏掉的手臂抬起动作，同时把靠近直接当成了接触。抬起、前伸、靠近、接触、按压是五个阶段，画面证据支持前三步，第四步是模型的推断。

## 常见问题

### 有思考输出但没有最终回答

总输出上限（-n）和思考预算（--reasoning-budget）设成一样的值时，token 在思考阶段就用完了，最终回答没有空间写出来。-n 设得比 --reasoning-budget 大，给回答留余量就行。

### llama-mtmd-cli 不认 reasoning 参数

llama-mtmd-cli 不接受 reasoning-effort、reasoning-budget、reasoning-format，加了会在模型加载前直接退出。只用 llama-mtmd-cli --help 里列出的参数。

### 帮助里有 --video，能跑原生视频吗

GLM5Next 源码里明确写了

    glm5next spells video with its own token pair, but video is not supported here

模型专属的视频 token 还没接入。--video 走的是通用媒体路径的拆帧，不是 GLM-5.3-Flash 的原生视频处理。

## 小结

四张 R9700 以 Vulkan layer split 加载了 UD-Q2_K_XL 量化的 GLM-5.3-Flash，文本端跑通了短回答和 7266-token 检索，静态图片返回了正确内容。视觉模块的部分算子目前回退到 CPU，原生视频处理还没接入，这两块等 GLM5Next 合并主线后值得重新验证。

有在其他 Radeon 上试过这个模型的吗？什么卡、几张、用的哪个量化，评论区聊聊。

## 参考资料

- [GLM-5.3-Flash 官方模型卡](https://huggingface.co/zai-org/GLM-5.3-Flash)
- [Unsloth GLM-5.3-Flash GGUF](https://huggingface.co/unsloth/GLM-5.3-Flash-GGUF)
- [llama.cpp GLM5Next 支持 PR](https://github.com/ggml-org/llama.cpp/pull/27754)
- [GLM5Next 视频限制源码](https://github.com/unslothai/llama.cpp/blob/949f7efb097eb20ef36fecdb1afaebff9a4ae7ed/tools/mtmd/mtmd.cpp#L863-L869)
