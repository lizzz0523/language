## 常见指标的javascript实现(KDJ篇)

KDJ指标，又叫随机指标或随机振荡器指标（Stochastic Oscillator），属于超买超卖指标中的一种，由George C. Lane在1950年代推广使用。“随机”一词是指价格在一段时间内相对于其波动范围的位置。

要计算KDJ指标，首先需要定义出某个基于时间的数值序列`X`的波动范围。KDJ采用的是类似唐奇安通道（Donchian channel）的计算方法，把过去`N`个时间单位内的最大值和最小值作为`X`当前的波动范围。对于股票的每日收盘价，就是其过去`N`天的最高价和最低价。并由此计算出一个原始随机值（Raw Stochastic Value）：
```
HIGH[i] = max(X[i-N:i])
LOW[i] = min(X[i-N:i])
RSV[i] = (X[i] - LOW[i]) / (HIGH[i] - LOW[i]) * 100
```
这个原始随机值，代表的就是`X`当前的值，在整个波动范围中的相对位置（0是最低位，100是最高位）。

第二步是要根据`RSV`的值计算出`K`和`D`的值，首先我们看看计算`K`和`D`的值的递推公式：
```
K[i] = (1 - A) * K[i - 1] + A * RSV[i]
D[i] = (1 - A) * D[i - 1] + A * K[i]
```
怎么样，有没有觉得很眼熟，没错，上面这个公式和计算EMA的公式是一样的，也就是说这里的`K`值实际就是`RSV`值的EMA，而`D`值就是`K`值的EMA。

最后，我们就可以根据`K`，`D`的值计算出`J`值，其公式为：
```
J[i] = 3 * K[i] - 2 * D[i] = K[i] + 2 * (K[i] - D[i])
```
实际就是在`K`值的基础上叠加了一个`K`，`D`值的一个差分计算，起到一个对`K`的放大作用。怎么样，是不是也很熟悉？没错，KDJ这套计算的方法和MACD的计算方法如出一辙，`K`就是MACD中的`DIF`，`D`就是MACD中的`DEA`，而`J`就类似MACD中的`MACD`。考虑到KDJ指标出现的更早，估计应该是MACD参考了这里的计算方式。

下面是采用javascript实现的，计算一个收盘价（随机生成）的9天KDJ：
```javascript
const close = Array.from({ length: 100 }, () => 100 * Math.random());
const period = 9;
const emaperiod = 3;
```
```javascript
function max(arr, start, end) {
  return Math.max(...arr.slice(start, end))
}

function min(arr, start, end) {
  return Math.min(...arr.slice(start, end))
}

function ema(arr, period) {
  const a = 2 / (period + 1);
  const b = 1 - a;
  for (let i = 0; i < arr.length; i++) {
    ema[i] = b * (ema[i - 1] || 0) + a * arr[i];
  }
  return ema;
}

function add(arr1, arr2, times = 1) {
  const add = [];
  for (let i = 0; i < arr1.length; i++) {
    add[i] = times * (arr1[i] + arr2[i]);
  }
  return add;
}

function diff(arr1, arr2, times = 1) {
  const diff = [];
  for (let i = 0; i < arr1.length; i++) {
    diff[i] = times * (arr1[i] - arr2[i]);
  }
  return diff;
}

const rsv = [];
for (let i = 0; i < close.length; i++) {
  const high = max(close, i - period + 1, i + 1);
  const low = min(close, i - period + 1, i + 1);
  rsv[i] = (close[i] - low) / (high - low) * 100;
}
const k = ema(rsv, emaperiod);
const d = ema(k, emaperiod);
const j = add(k, diff(k, d, 2));
```
这里和MACD计算一样，可以把函数调用，改为单个循环：
```javascript
const rsv = [];
const k = [];
const d = [];
const j = [];
const a = 1 / emaperiod;
const b = 1 - a;
for (let i = 0; i < close.length; i++) {
  const high = max(close, i - period + 1, i + 1);
  const low = min(close, i - period + 1, i + 1);
  rsv[i] = (close[i] - low) / (high - low) * 100;
  k[i] = b * (k[i - 1] || 0) + a * rsv[i];
  d[i] = b * (d[i - 1] || 0) + a * k[i];
  j[i] = k[i] + 2 * (k[i] - d[i]);
}
```
当然这里在计算最大最小值，也可以引入一些高效的数据结构，来加快计算的速度，这里就暂时不展示了（我也不懂，哈哈哈）。

接下来我们可以讨论一下KDJ的意义。由于KDJ的计算方法和MACD一样，所以KDJ本质上也是计算一种趋势，或者更确切的说是`RSV`的趋势。首先`K`值是`RSV`的EMA，这里可以理解为对`RSV`中的滤波处理，提取其主要成分，过滤高频噪声。接下来`D`值是`K`的EMA，这里和MACD中的`DIF`和`DEA`实际上是对等的，就是求出快慢两个MA（`K`可以理解为`EMA(K, 1)`。最后通过在`K`上叠加一个`K`，`D`的差分值得到`J`，相比于`K`值，由于`J`值叠加了一个和当前趋势相同的波幅，使得其更加的敏感。

那KDJ要如何使用的，这里也是和MACD的一样，当`K`，`D`交叉时，意味着趋势反转，可以作为买卖信号。但和MACD一样，KDJ中也使用到MA的计算，同样有时间滞后的问题，不过由于KDJ在计算时使用的`N`较小（一般为3），使得滞后不明显，但也意味着KDJ保留了大量的噪声，这也是KDJ名称中“随机”两个字的由来，表示价格在波动期间，这种趋势反转信号有可能会“随机”出现。所以大多数的买卖信号，都是假信号。

但KDJ本身除了反映趋势变化之外，还有一个有用的信息，那就是超买超卖信息，当`K`，`D`，`J`均出现在超买区（`K`，`D`大于80，`J`大于100）或者超卖区（`K`，`D`小于20，`J`小于0）时，意味着当前价格逼近近期的最高价或者最低价，伴随着阻力或者支撑的效应，而如果这时再出现`K`，`D`交叉的买卖信号，那么这个信号就有很大的可能性是趋势反转的点，就可以进行买卖了。

以上