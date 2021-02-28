## 常见指标的javascript实现(MA篇)

MA是移动平均（Moving Average）的意思，是对某一个基于时间的数值序列`X`，计算其过去`N`个时间单位内的平均值`A`。例如对于股票的每日收盘价，其`MA`值等于其过去`N`天的平均收盘价。其主要作用是希望找到数值序列`X`的一个（稳定的）主要成分，类似于信号处理里的滤波处理，过滤掉高频噪声。而对于不同的MA指标（例如SMA，EMA）主要是计算平均值的算法不一样。

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
再进一步，换成递推形式（`SMA[i] = f(SMA[i - 1]) = SMA[i - 1] + A * (X[i] - X[i - N])`）：
```javascript
const sma = [];
const a = 1 / period;
for (let i = 0; i < close.length; i++) {
  sma[i] = (sma[i - 1] || 0) + a * (close[i] - (close[i - period] || 0));
}
```

### EMA（指数移动平均，Exponential Moving Average）
相对于SMA的简单算术平均（认为过去每一个时间单位上的值对当前值的影响是相同的），EMA则采用的是加权平均（认为离当前时间越近的值，对当前值的影响越大）。
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
再进一步，换成递推形式（`EMA[i] = f(EMA[i - 1]) = (1 - A) * EMA[i - 1] + A * X[i]`）：
```javascript
const ema = [];
const a = 2 / (period + 1);
const b = 1 - a;
for (let i = 0; i < close.length; i++) {
  ema[i] = b * (ema[i - 1] || 0) + a * close[i];
}
```

### MACD（移动平均聚散度，Moving Average Convergence Divergence）
在知道了MA是什么后，我们就可以利用MA来计算一个重要的技术指标 -- MACD。

MACD是移动平均聚散度（Moving Average Convergence Divergence）的意思。是用来描述快慢两个MA值之间的差异程度的，由Gerald Appel于1970年代提出，用于研判股票价格变化的趋势。

从上面的介绍中，我们知道`N`越小的MA，对`X`的当前值反应越快，越能代表`X`最近的值，而`N`越大的MA，对`X`的当前值的反应较慢，也就是越能代表`X`过去的值，而如果我们想知道相对于过去，`X`最近的值是涨了还是跌了，涨跌的强度如何，我们就可以用两个`N`不同的MA值进行差分处理，如果快的MA比慢的MA要大，证明`X`的值在上涨，并且差值越大，涨得越猛，相反如果快的MA比慢的MA要小，证明`X`的值在下跌，而且差值（负的）越小，跌得越重。

换句话说，快慢两个MA值之间的差异程度，能反映出`X`的变化趋势。而MACD指标就是用来计算差异程度的：
```
MA1 = EMA(X, N), MA2 = EMA(X, M)
DIF = MA1 - MA2, DEA = EMA(DIF, K)
MACD = DIF - DEA
其中: K < N < M
```
从上面的公式可以看到，我们首先需要计算两个MA值，一个是快的`MA1`（`N`比较小的），另一个是慢的`MA2`（`N`比较大的）。接下来进行差分计算`DIF`。然后再一次对`DIF`进行MA计算得出`DEA`，这时候你就得到另外两个快慢MA值`DIF`和`DEA`（`DIF`可以理解为`EMA(DIF, 1)`）。最后再对这两个MA值进行差分计算，得到`MACD`值。

下面是采用javascript实现的，其中`K=9`，`N=12`，`M=26`：
```javascript
const close = Array.from({ length: 100 }, () => 100 * Math.random());
const fast = 12;
const slow = 26;
const signal = 9;
```
```javascript
function ema(arr, period) {
  const a = 2 / (period + 1);
  const b = 1 - a;
  for (let i = 0; i < arr.length; i++) {
    ema[i] = b * (ema[i - 1] || 0) + a * arr[i];
  }
  return ema;
}

function diff(arr1, arr2) {
  const diff = [];
  for (let i = 0; i < arr1.length; i++) {
    diff[i] = arr1[i] - arr2[i];
  }
  return diff;
}

const ma1 = ema(close, fast);
const ma2 = ema(close, show);
const dif = diff(ma1, ma2);
const dea = ema(dif, signal);
const macd = diff(dif, dea);
```
这里采用的是函数调用的方式来计算，也可以直接展开的单个循环：
```javascript
let ma1 = 0;
let ma2 = 0;
const dif = [];
const dea = [];
const macd = [];
const fastA = 1 / fast;
const fastB = 1 - fastA;
const slowA = 1 / slow;
const slowB = 1 - slowA;
const signalA = 1 / signal;
const signalB = 1 - signalA;
for (let i = 0; i < close.length; i++) {
  ma1 = fastB * ma1 + fastA * close[i];
  ma2 = slowB * ma2 + slowA * close[i];
  dif[i] = ma1 - ma2;
  dea[i] = signalB * (dea[i - 1] || 0) + signalA * dif[i];
  macd[i] = dif[i] - dea[i];
}
```
其中我们上面说到的反映`X`的变化趋势的是`DIF`和`DEA`，而`MACD`本身，反映的是变化趋势自身的变化趋势。你可以把`X`，`DIF`和`MACD`看作是路程，速度和加速度的关系。当加速度为正的时候，速度会往正的方向走，原来为负的速度会递减，原来为正的速度会递增，最终产生正的路程。同样，当`MACD`为正的时候，`DIF`就会产生由负转正的趋势，最终导致`X`的拉升。

有一些说法，当`MACD`值穿过0点时，意味着趋势反转，可以视为买卖信号，但由于MA计算带来的时间滞后性，一般这个买卖点，都会比实际的转接点晚几天，所以MACD指标一般是作为趋势信号来辅助其他指标进行分析的，而比较少直接用来作为买卖信号。

以上