# AMD显卡如何跑PyTorch？一篇搞定

## 标题

AMD显卡如何跑PyTorch？一篇搞定

## 正文

“好多人抱怨：A卡实在太折腾，装个ROCm、装个PyTorch都费劲。”

这段时间听到不少人这么说😂

AMD显卡可以装PyTorch, 但你需要知道，

大部分问题其实不是命令错了。

而是这几个版本没对上：

✅ 显卡对应的 gfx

✅ ROCm 版本

✅ PyTorch 版本

✅ Python 版本

只要其中一个不匹配，就可能出现：

❌ GPU识别失败

❌ torch报错

❌ 安装成功但无法调用显卡

🧹 安装前先别急着复制命令

先查清楚显卡对应的gfx，再确认系统和Python版本。之前装过其他版本的话，建议先把旧包卸干净，避免不同来源的包打架。

🐧 Linux有5条路可以选

其实现在已经有好几种安装方式。

不同场景对应不同方案。

👉 显卡gfx明确

选 AMD Multi-Arch

👉 想尝鲜新功能

选 Nightly

👉 不想污染本机环境

直接 Docker

👉 已经确定ROCm版本

选 AMD manylinux

👉 想严格按版本矩阵来

选 PyTorch ROCm 7.2源

具体命令和版本对应关系都整理到了配图里。

建议直接保存，后面重装基本都会用到。

🪟 Windows用户看这里

目前支持列表内的部分Radeon显卡和Ryzen处理器，已经可以在Windows原生安装PyTorch。

安装前需要对好显卡、Windows、驱动和Python版本，再依次安装ROCm组件及PyTorch wheels。

不过要注意：目前是PyTorch可以原生运行，不代表整套ROCm软件栈都能在Windows使用。设备暂时不在支持范围内，可以再看看WSL2方案。

✅ 装完先跑这条命令

python -c "import torch; print(torch.__version__, torch.cuda.is_available())"

看到版本号和True，说明PyTorch已经识别到AMD GPU。

ROCm版仍然沿用torch.cuda接口，所以看到cuda不用怀疑，跑的还是A卡。

再碎碎念一句：gfx要对应，三个PyTorch包别乱配，不同ROCm发布线也别混装。很多“装完不认卡”，问题就出在这里。

你现在是在Windows还是Linux上折腾PyTorch？

在下面扣个W/L带上显卡型号，也可以说说自己装PyTorch的时候卡在哪一步🙋

告诉我们出现频率较高的问题，我们整理成下一篇排错教程。

我们来自AMD个人工程师团队，后面还会继续整理更多干货内容。
