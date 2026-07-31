# A 卡装 AMD ROCm｜gfx 代号别填错

- **平台**：小红书
- **类型**：基础内容（第 1 篇）
- **来源**：搬运开源项目 [hello-rocm](https://github.com/datawhalechina/hello-rocm)（Datawhale 社区），按小红书图文风格改写
- **数据依据**：ROCm 7.14.0 兼容性矩阵
- **配图**：`images/` 下 7 张 1080×1440 竖版图，文件名顺序即发布顺序

---

## 标题

**A卡装AMD ROCm｜gfx代号别填错**（20 字，小红书上限 20 字）

## 正文

A 卡想跑 AI，得先把环境装起来。

装 AMD ROCm AI 软件栈的时候，命令里那串 gfx1151 到底是什么？填错一个数字，装完 PyTorch 直接不认卡，一晚上白折腾。

【01 先分清两条产品线】

AMD 的 GPU 分两个家族。

CDNA 是数据中心那条线，也就是 Instinct MI 系列，MI300X、MI350X 都在这里，用来做大规模训练和推理。

RDNA 是消费级这条线，Radeon RX 游戏卡、Radeon PRO 工作站卡，还有 Ryzen AI 笔记本上的核显。

机房里跑的是 CDNA，你手上这台是 RDNA。

【02 gfx 是什么】

gfx1151 这种写法叫 LLVM Target，是编译器用来认显卡的编号。

架构名（RDNA 3.5）是给人看的，gfx 号是给编译器看的。装 AMD ROCm AI 软件栈、装 PyTorch wheel、编译 HIP 代码，填的都是它。

【03 速查表】

Radeon RX 和 PRO 看 P3，Ryzen AI 核显看 P4，数据中心的 Instinct 在 P5。

手里是 RX 6000 的注意一下：它属于 RDNA 2，不在 ROCm 支持列表里（RX 6800、6900 是 gfx1030），能不能跑要自己试。

【04 怎么查自己的卡】

一行命令：

rocminfo | grep gfx

输出里那串 gfx 号就是答案，见 P6。

【05 一个高频踩坑】

经常看到有人写 gfx3.5，没有这个写法。

RDNA 3.5 对应的是 gfx1150、gfx1151、gfx1152。安装命令里填 3.5，装完还是跑不起来。

把你的显卡型号留在下面，我告诉你该用哪个 gfx。

下一期讲 AMD GPU 和 NVIDIA GPU 的产品对应关系，A 卡 N 卡型号怎么对上。

内容参考开源项目 hello-rocm（Datawhale 社区），完整对照表放在置顶那条。

#AMD #ROCm #GPU #显卡 #深度学习 #AI #PyTorch #程序员 #人工智能

## 首评

小红书正文放外链会影响流量，链接统一走评论区首评。以下为首评原文：

完整架构对照表 + gfx 速查（含 ROCm 版本兼容矩阵）在这里：
github.com/datawhalechina/hello-rocm
想看哪个方向的教程，在下面告诉我～

## 配图

正文里的 P 几对应下表，上传顺序不能乱。

| 位置 | 文件 | 内容 |
|:---|:---|:---|
| P1 | `01-cover.png` | 封面 |
| P2 | `02-cdna-rdna.png` | CDNA 与 RDNA 两条产品线 |
| P3 | `03-radeon.png` | Radeon RX / PRO 速查表（含 RDNA 2 说明） |
| P4 | `04-ryzen-apu.png` | Ryzen AI 核显速查表 |
| P5 | `05-instinct.png` | Instinct 数据中心 GPU 速查表 |
| P6 | `06-command.png` | `rocminfo \| grep gfx` 真机截图 |
| P7 | `07-pitfall.png` | gfx3.5 误区 + 下期预告 |

来源标注在正文结尾和每张图底部各出现一次，图片被单独转发时也带着出处。

配图由脚本统一生成（1080×1440，3:4 竖版）。需要改文案或增删表格行时重新出图即可，不用手动修图，review 有意见直接提。
