# 没A卡也能跑大模型？云算力5步上手

## 标题

**没A卡也能跑大模型？云算力5步上手**

## 正文

想跑 DeepSeek、Qwen、PyTorch，或者体验ROCm，但又不想为了尝鲜专门买一张 AMD 显卡？

其实可以先用云端 AMD GPU，兑换到跑起来只要 5 步👇

🚀 5 步开启 AMD GPU

1️⃣ 登录 AMD AI Developer Program，进入「Profile」

2️⃣ 进入「我的权益」→「LQ」, 进入兑换活动页，选择「立即兑换」

3️⃣ 输入想兑换的小时数，选择「去兑换」直接完成兑换

4️⃣ 打开 AMD Developer Cloud 并登录账号

5️⃣ 创建 Template，启动实例并进入 Notebook。现在就可以使用 AMD GPU 了！

⏱️ 注意：额度按实例运行时间计算

从选择 Launch 开始，实例只要处于运行状态，就会持续消耗额度——即使没有执行任何任务。

Token Factory 还提供了一批公共模型 API，不启动实例也能调用，这部分不消耗 GPU 实例额度。

⚠️ 用完要释放实例

关闭网页、退出浏览器、断开 SSH，都不代表实例已经停止！

正确步骤是：

「Profile」→「Active Instance」→ 选择红色的「Destroy Instance」

这样才算真正停止计时。

平台页面和活动规则可能调整，请以实际显示为准。

如果现在免费给你AMD GPU 云算力，你会先跑什么？

DeepSeek？
Qwen？
ComfyUI？
FLUX？
还是其他模型？

欢迎在下面👇说说～ 后续我们也会参考大家的问题继续做实测。

我们是 AMD 个人工程师团队，后续还会继续分享 ROCm 环境搭建、GPU 选型和本地部署大模型等内容。
