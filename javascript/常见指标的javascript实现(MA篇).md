## 常见指标的javascript实现(MA篇)

MA是移动平均（Moving Average）的意思，是对某一个基于时间的数值序列`X`，计算其过去`N`个时间单位内的平均值`A`。例如对于股票的每日收盘价，其MA值等于其过去N天的平均收盘价。其主要作用是希望找到数值序列`X`的一个变化趋势，类似于信号处理里的滤波处理，过滤掉高频噪声。而对于不同的MA指标（例如SMA，EMA）主要是计算平均值的算法不一样。

### SMA（简单移动平均，Simple Moving Average）
对于SMA，其计算平均值的方法，就是简单的算术平均
```
SMA[i] = A * (X[i] + X[i - 1] + ... + X[i - N + 1])
其中: A = 1 / N
```
下面是采用javascript实现的，计算一个收盘价（随机生成）的6天简单移动平均值：
```javascript
const close = Array.from({ length: 100 }, () => 100 * Math.random());
const period = 6;
```
```javascript
const sma = [];
const a = 1 / period;
for (let i = 0; i < close.length; i++) {
  let sum = 0;
  for (let j = 0; j <= period - 1; j++) {
    sum += (close[i - j] || 0);
  }
  sma[i] = a * sum;
}
```
但这里在计算平均值时，需要重复的进行求和计算，而实际上，我们仅仅需要对`sum`加上当前的数值，并减去”过期“的数值来替代内循环：
```javascript
const sma = [];
const a = 1 / period;
let sum = 0;
for (let i = 0; i < close.length; i++) {
  sum = sum + close[i] - (close[i - period] || 0);
  sma[i] = a * sum;
}
```
再进一步，换成递推形式（SMA[i] = f(SMA[i - 1])）：
```javascript
const sma = [];
const a = 1 / period;
for (let i = 0; i < close.length; i++) {
  sma[i] = (sma[i - 1] || 0) + a * (close[i] - (close[i - period] || 0));
}
```

### EMA（指数移动平均，Exponential Moving Average）
相对于SMA的简单算术平均（认为过去每一个时间单位上的值对当前值的影响是相同的），EMA则采用的是加权平均（认为离当前时间越近的值，但当前值的影响越大）。
```
EMA[i] = A * ((1 - A)^0 * X[i] + (1 - A)^1 * X[i - 1] + ... + (1 - A)^i * X[0])
其中: A = 2 / (N + 1)
```
下面是采用javascript实现的，计算一个收盘价（随机生成）的12天指数移动平均值：
```javascript
const close = Array.from({ length: 100 }, () => 100 * Math.random());
const period = 12;
```
```javascript
const ema = [];
const a = 2 / (period + 1);
const b = 1 - a;
for (let i = 0; i < close.length; i++) {
  let sum = 0;
  for (let j = 0; j <= i; j++) {
    sum += b ** j * close[i - j];
  }
  ema[i] = a * sum;
}
```
同样，我们可以把计算`sum`的过程简化，来消除内循环：
```javascript
const ema = [];
const a = 2 / (period + 1);
const b = 1 - a;
let sum = 0;
for (let i = 0; i < close.length; i++) {
  sum = b * sum + close[i];
  ema[i] = a * sum;
}
```
再进一步，换成递推形式（EMA[i] = f(EMA[i - 1])）：
```javascript
const ema = [];
const a = 2 / (period + 1);
const b = 1 - a;
for (let i = 0; i < close.length; i++) {
  ema[i] = b * (ema[i - 1] || 0) + a * close[i];
}
```

以上