本实验对尺寸较大的遥感图像进行分块处理，实验原理基于实验五的IHS遥感图像融合。

在代码中通过设定不同的参数使用deal函数对图像进行分块，同时使用clock函数记录运行时间。

运行结果如下：

（15696/256=62）

![image](https://github.com/SE-superGroup/week6/blob/master/01.PNG)

时间显示如下：

![image](https://github.com/SE-superGroup/week6/blob/master/00.PNG)

256*256分块方式占据了程序的主要运行时间，imgXlen*256的时间则短得多。

