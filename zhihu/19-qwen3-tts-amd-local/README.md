# 给Qwen3-TTS一段约10秒参考声，R9700本地生成中文配音

<video controls src="media/hero-reference-vs-clone-v3.mp4"></video>

先听上面两段声音。前面10秒是一段录音。我让AI从这段录音里学了声音，用它说了后面那段新内容。跑在一张R9700上，暖机后不到2秒就生成好了。(PS:如果你没有本地的显卡，想免费使用下怎么办？欢迎来Radeon Cloud！注册一个账号，申请云环境，也可以跑下面的命令，详见：[gpu算力租赁哪个平台值得推荐?](https://www.zhihu.com/question/539158346/answer/2071287480968536065))

这次用的模型是Qwen3-TTS，阿里最近开源的语音模型，1.7B参数，需要两份权重、加起来大约只有2.4GB，跑起来大约占4到6GB显存。

Qwen3-TTS的Base版本支持声音克隆。给一段约10秒的声音样本，加上对应的文字转写，就能用这个声音说新的内容。上面的样本来自它自带的一个预置音色，也可以换成自己的录音。

## 从空目录到第一段克隆声音

Qwen3-TTS的官方实现是PyTorch，但是PyTorch和ROCm，在消费卡这边往往有小问题，所以社区做了一个叫[qwentts.cpp](https://github.com/ServeurpersoCom/qwentts.cpp)的C++推理引擎。用的是GGML（就是llama.cpp底下那套C++推理框架），支持Vulkan。Vulkan稳多了。

所以我们用qwentts.cpp。用Docker跑，拉一个ROCm基础镜像当容器环境，里面AMD驱动和Vulkan支持都装好了，省得自己配。

```bash
docker run -d --name qwen-tts \
  --device /dev/kfd --device /dev/dri \
  --group-add $(getent group video | cut -d: -f3) \
  --group-add $(getent group render | cut -d: -f3) \
  --shm-size 8g \
  -v $HOME/qwen3-tts:/workspace \
  rocm/pytorch:rocm7.2.4_ubuntu24.04_py3.12_pytorch_release_2.10.0 \
  sleep infinity
```

一些参数的意思：
- `--device /dev/kfd --device /dev/dri` 把AMD显卡设备传进容器
- `--group-add` 后面那串取的是宿主机video和render组的数字编号，给容器GPU访问权限
- `-v $HOME/qwen3-tts:/workspace` 把本地目录映射进去，容器里的`/workspace`就是外面的`$HOME/qwen3-tts`
- `sleep infinity` 让容器一直开着，后面编译、下模型、跑生成都在里面做，不用反复开关

然后进容器。

```bash
docker exec -it qwen-tts bash
```

进去以后装编译工具链，拉qwentts.cpp源码，编译Vulkan后端。

```bash
apt-get update && apt-get install -y \
  git cmake ninja-build build-essential \
  libvulkan-dev mesa-vulkan-drivers glslc libshaderc-dev

git clone --recursive https://github.com/ServeurpersoCom/qwentts.cpp.git /workspace/qwentts.cpp
cd /workspace/qwentts.cpp
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON
cmake --build build -j$(nproc) --target qwen-tts qwen-codec
```

（PS: 我跑的时候qwentts.cpp最新提交是`177925e`，直接clone拉最新的一般没问题，遇到报错可以退回这个版本。）

编译完了，下模型。Qwen3-TTS用两份权重，分别叫Talker和Tokenizer。加起来约2.4 GB。

```bash
mkdir -p /workspace/models
curl -fL -o /workspace/models/qwen-talker-1.7b-base-Q8_0.gguf \
  "https://huggingface.co/Serveurperso/Qwen3-TTS-GGUF/resolve/main/qwen-talker-1.7b-base-Q8_0.gguf"
curl -fL -o /workspace/models/qwen-tokenizer-12hz-Q8_0.gguf \
  "https://huggingface.co/Serveurperso/Qwen3-TTS-GGUF/resolve/main/qwen-tokenizer-12hz-Q8_0.gguf"
```

这里额外解释下这两个文件在干什么。
- Talker是主模型，1.7B参数，输出的是音频编码。每秒只做12.5个决策，每个决策代表一小段声音，文件名里的"12Hz"就是这个意思。12.5个token就够表示一秒的声音，R9700才跑得出6倍实时的速度。给它参考声的编码和一个只有8 KB的声纹文件，它就能用同样的声音写出新的音频编码，像根据一个人的声音写一份新曲谱。
- Tokenizer其实是个音频编解码器（和文本分词器没关系），把Talker写的曲谱变成你能听到的WAV。它内部用16层编码表示声音，第一层管内容和大致音高，后面的层逐步补上音色和气息的细节。

跑生成之前还需要三个文件：一段约10秒的参考声（reference.wav）、对应的转写（ref.txt）、和你想让它说的新文案（prompt.txt）。

ref.txt里写的是参考声的转写，参考声说了什么就写什么，最好一字不差。多了或者少了不会报错，但生成出来的开头可能冒出一段奇怪的尾音。

把它们放进`$HOME/qwen3-tts/`就行。

```bash
/workspace/qwentts.cpp/build/qwen-tts \
  --device Vulkan0 \
  --model /workspace/models/qwen-talker-1.7b-base-Q8_0.gguf \
  --codec /workspace/models/qwen-tokenizer-12hz-Q8_0.gguf \
  --ref-wav /workspace/reference.wav \
  --ref-text /workspace/ref.txt \
  --lang Chinese \
  -o /workspace/clone.wav \
  < /workspace/prompt.txt
```

跑完后，clone.wav就在/workspace底下。到这里就算跑通了。

## 约10秒参考声怎么准备

官方说约3秒就能克隆，不过根据我们的实测，用8到12秒效果会更加稳定。这是因为编码器不会选择性过滤声学特征，音色、节奏、背景噪音都会识别处理，而稍长的素材才能有这些内容。

用手机自带的录音App就行，iPhone上叫语音备忘录，安卓一般叫录音。电脑上可以装Audacity，免费开源的。

qwentts.cpp要24000 Hz单声道WAV。手机录的m4a用格式工厂或者ffmpeg转一下就行。

```bash
ffmpeg -i input.m4a -ar 24000 -ac 1 reference.wav
```

## R9700上的生成速度

拿R9700实际跑了几段不同长度的文本。10秒的克隆语音，1.66秒生成完成，短句基本秒出。

下面这段是22秒的中文旁白，3.6秒跑完。

<video controls src="media/medium-narration-v3.mp4"></video>

我还试了一段接近2分钟的长文配音，19秒跑完。

<video controls src="media/long-narration-v3.mp4"></video>

测了下性能，同一张卡上Vulkan比HIP快约18%。

661个字的文本一次性生成能跑完。但按句分段更可靠，哪句不满意单独重做就行。不过，分段的代价是片段之间可能有音量变化。

qwentts.cpp还自带一个OpenAI兼容的本地API，注册一次声音后面只传文本，可以批量做旁白，不需要一条条命令。


## 看看你的需求？

本地TTS能做的事比念一句话多得多。视频配音、播客、有声书、游戏角色，甚至接上Whisper做语音翻译的前半段。你打算拿它做什么？

评论区聊聊。手里有其他卡的，试试跑一遍贴个时间？看看不同卡差多少。

## 参考资料

- [Qwen3-TTS 官方仓库](https://github.com/QwenLM/Qwen3-TTS)
- [Qwen3-TTS 1.7B Base 模型](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-Base)
- [qwentts.cpp](https://github.com/ServeurpersoCom/qwentts.cpp)
- [qwentts.cpp GGUF 权重](https://huggingface.co/Serveurperso/Qwen3-TTS-GGUF)

#AMD #R9700 #Qwen3TTS #声音克隆 #TTS #本地AI
