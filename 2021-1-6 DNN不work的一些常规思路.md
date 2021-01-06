2021-1-6 DNN不work的一些常规思路

在网络结构和损失函数没有大问题的前提下，通常有以下五个思路：

underfiting:激活函数，学习率

overfting:earlystop，regurlazition，dropout

激活函数：

sigmoid优点：便于入门演示教学

sigmoid缺点：因为函数的特性，导数变化越来越小，经过计算后的梯度更新会很小，出现gradient vanish.

且计算因为需要带e，较为复杂。

relu：既然sigmoid的梯度消失是因为函数值被强行挤压在[0,1]区间，那么将区间完全释放出来。



Adam 就是RMSprop+monentum 

前者带有历史梯度和当前梯度的加权

后者是历史梯度惯性的描述。



L1和L2正则。

L1绝对值，定长增减梯度，大的梯度变化可能影响较少，小的较大。

L2平方和开根号，百分比衰减，大的梯度变化较大，小的较小。