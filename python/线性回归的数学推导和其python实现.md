
# 线性回归的数学推导和其python实现

e为真实值和预测值之间的误差

$$ e = y - h(x,W) $$

而这个误差e，是由许多不同的因素而产生的，例如：
* 可能我们所使用到的特征数量不足
* 可能收集的数据带有噪声

根据中心极限定理，当一个变量是大量的独立随机事件产生的结果，那么我们倾向于使这个变量服从高斯分布，即：

$$ P(e)\sim N(0, \sigma) $$

即：

$$ P(e)=\frac{1}{\sqrt{2\pi}\sigma}\exp(\frac{-e^2}{2\sigma^2}) $$

把误差e的定义带入上式得：

$$ P(y|x;W)=\frac{1}{\sqrt{2\pi}\sigma}\exp(\frac{-(y-h(x,W))^2}{2\sigma^2}) $$

可以看出， $ P(y|x;W) $ 亦服从高斯分布：

$$ P(y|x;W)\sim N(h(x,W),\sigma) $$

为了使我的假设的模型尽量和真实的情况相符，亦即求出模型的最大似然：

$$ \max \prod_{i=1}^{n} P(y_i|x;W_i) $$

令 $ L(W) $ 为模型关于参数 $ W $ 的似然函数：

$$ L(W)=\prod_{i=1}^{n} P(y_i|x;W_i) $$

$$ l(W)=\ln(L(W)) $$

$$ l(W)=\ln(\prod_{i=1}^{n} P(y_i|x;W_i))=\ln(\prod_{i=1}^{n} \frac{1}{\sqrt{2\pi}\sigma}\exp(\frac{-(y_i-h(x,W_i))^2}{2\sigma^2})) $$

$$ l(W)=\sum_{i=1}^{n} \ln(\frac{1}{\sqrt{2\pi}\sigma}\exp(\frac{-(y_i-h(x,W_i))^2}{2\sigma^2}) $$

$$ l(W)=\sum_{i=1}^{n} \ln(\frac{1}{\sqrt{2\pi}\sigma})+ln(\exp(\frac{-(y_i-h(x,W_i))^2}{2\sigma^2}) $$

$$ l(W)=n\ln(\frac{1}{\sqrt{2\pi}\sigma})+\sum_{i=1}^{n}\ln(\exp(\frac{-(y_i-h(x,W_i))^2}{2\sigma^2})) $$

$$ l(W)=n\ln(\frac{1}{\sqrt{2\pi}\sigma})+\sum_{i=1}^{n} \frac{-(y_i-h(x,W_i))^2}{2\sigma^2} $$

到这里，我们可以得出，最多似然，实际就是求模型的最小二乘参数：

$$ \max L(W)\sim \max l(W)=\max n\ln(\frac{1}{\sqrt{2\pi}\sigma})+\sum_{i=1}^{n} \frac{-(y_i-h(x,W_i))^2}{2\sigma^2}\sim\min \sum_{i=1}^{n}(y_i-h(x,W_i))^2 $$

最小二乘 $ \min \sum_{i=1}^{n}(y_i-h(x,W_i))^2 $ ，采用梯度下降算法求解，设 $ Q $ 为最小二乘：

$$ Q=\sum_{i=1}^{n}\frac{1}{2}(y_i - h(x,W_i))^2 $$

其梯度表达式为：

$$ \frac{\partial Q}{\partial W_i}=-(y_i-h(x,W_i))x_i $$

因此我们可以得出参数 $ W $ 的更新方程为：

$$ W^k_i=W^{k-1}_i-a\frac{\partial Q}{\partial W_i} $$

其中 $ a $ 为学习率，一下为线性回归的python实现：


```python
import numpy as np

# y的列向量
y = np.mat([
    [7],
    [6],
    [6],
    [-5],
    [-9]
]).T # 1,5

# x的列向量
x = np.mat([
    [1,2,1],
    [2,2,1],
    [3,1,1],
    [7,8,1],
    [9,10,1]
]).T # 2,5

# 权重
W = np.mat([1.0,1.0,1.0]) # 1,2

# 迭代次数
niter = 10000
# 学习率
lr = 0.005

for i in range(niter):
    W = W + lr * (y - W * x) * x.T
    
print(W)
```

    [[-1. -1. 10.]]
    


```python
    
```
