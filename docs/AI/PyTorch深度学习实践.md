# PyTorch深度学习实践

## 资料
* [视频：刘二大人 PyTorch深度学习实践](https://www.bilibili.com/video/BV1Y7411d7Ys?spm_id_from=333.788.player.switch&vd_source=2a33d03ec3e67e46971208a7faa0dcda&p=2) （10 CNN看完了，可以再看一遍）

## 机器学习基本概念简介
1. 数据集
    - 训练集 Train：模型在这里学习权重、梯度更新，学会数据特征。（平时的练习题）
    - 开发集 Dev（验证集）：用来对比多个模型、调整学习率、层数、batch‑size、正则、激活函数等超参数。（模拟考，你根据模拟考成绩调整你的学习方法、刷题策略）
    - 测试集 Test：只在全部调试结束之后，运行一次，查看模型面对从未见过数据的泛化能力。（正式高考）

    为什么不能直接拿测试集调参？

    如果你反复看测试集准确率，然后修改模型、超参数去适配测试集，模型会间接记住测试集的特征，相当于你把测试集当成了训练集。最后测试集的精度会虚高，无法代表陌生真实数据的表现，出现数据泄露、过拟合测试集。

2. NLL Loss和CrossEntropy Loss的区别：

    - NLL Loss: Negative Log Likelihood Loss，用于分类任务，计算每个样本的负对数似然值。
    - CrossEntropy Loss: 交叉熵损失，用于分类任务，计算每个样本的负对数似然值，然后取平均。
    - 总结：CrossEntropy Loss = LogSoftmax + NLL Loss。CrossEntropy Loss内部会先对模型输出进行LogSoftmax操作，再计算NLL Loss。

    ![Softmax_Classifier](../files/AI/Lecture_09_Softmax_Classifier.pdf){ type=application/pdf style="width:100%; min-height:70vh;" }
