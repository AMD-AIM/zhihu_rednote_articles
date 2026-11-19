标题：Windows+AMD训练YOLO，真的跑起来了！

正文：
以后在Windows上训练YOLO，AMD显卡也能上了！

以前是识别能跑，一开训练就报错。
这道坎，AMD的工程师修好了，修复已经合进ROCm的库（PR #13437）。
YOLO的训练代码一行不用改，在Radeon 8060S上已经完整跑通。

📦 YOLO这边也接上了
10月1日，Ultralytics也合入了AMD的支持（PR #24137）：
· Linux上导出的ONNX模型，会自动用AMD显卡加速推理
· 文档补上了AMD显卡的训练和部署说明
pip装8.4.171及以后的版本就有。

你拿YOLO做过什么？想在A卡上训练点什么？
在Windows上踩过坑的，把报错发出来，我们帮你看看👇