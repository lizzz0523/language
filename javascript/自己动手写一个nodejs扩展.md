## 自己动手写一个nodejs扩展

最近这段时间，在学习投资理财时，无意中接触到[ta-lib](https://ta-lib.org/)这个工具库。ta-lib是一个用于计算技术指标的库，使用c编写，性能好，并且能在python中（通过[python扩展](https://github.com/mrjbq7/ta-lib)）直接调用。

但可惜自己的python写得不算太溜，一般的脚本工具还行，但如果要编写有一定复杂度的应用程序，还是js比较顺手。于是就想看看npm这边有没有ta-lib的node包，果然有热心的小伙伴已经实现了一个（[node-talib](https://www.npmjs.com/package/talib)）。非常好，也非常强大。

不过作为好奇宝宝，还是忍不住要翻翻源码，这个node包主要是通过，在使用c++编写的node扩展中调用ta-lib相关函数来实现的，看起来不算太复杂。而自己之前也没有正儿八经的编写过node扩展，这个轮子正好可以练练手。

理想很丰满，现实却打了你一把掌。要编写node扩展，首先是要懂一点c++，我的c++虽然写得不多，如果只是实现一个hello world扩展还是没有太大问题的。但问题是，除了要编写node c++相关代码，我还需要知道如何在c++中调用ta-lib的方法，然而ta-lib的文档比我想象中的要复杂，加上又是自己不熟悉的语言，出师不利。

不过，我并不想放弃这次练手的机会，于是，在浏览npm的过程中，我找到了另一个和ta-lib功能一样的库 -- [tulip](https://tulipindicators.org/)，这个库同样是采用c语言编写，从官网上提供的基准测试来看，性能也非常不错。而最最重要的是，tulip的文档非常简单，加起来就只有一页，没有太多学习成本。OK，就你了。

### 环境搭建
要编写node扩展，你至少需要两样东西：

 1. 需要一个编译工具，把你写的c++代码编译成.node文件
 2. 需要一套由node提供的，能与V8引擎交互的编程接口

对于编译工具，这个比较简单，node官方提供了一个名为[node-gyp](https://www.npmjs.com/package/node-gyp)的包，使用这个包提供的命令行工具，就能直接把你的c/c++代码编译成.node文件。我们可以直接使用npm来安装这个包：

```shell
$ npm install node-gyp
```

此外，在不同的平台下，你还需要安装相应的工具，node-gyp才能正常工作，例如我在macos下，就需要安装：

 1. Python 2.7, v3.5, v3.6, v3.7, 或者 v3.8
 2. XCode Command Line Tools

安装的过程也十分简单，macos自带Python，无需额外安装，而XCode Command Line Tools，只需在命令行上输入：

```shell
$ xcode-select --install
```

回车（可能还需要点击一些弹窗的确定按钮），等待数分钟即可安装完成。如果之前就安装过XCode Command Line Tools的小伙伴，在启动node-gyp时提示：

```shell
gyp: No Xcode or CLT version detected!
```

那可能是macos升级导致的问题，你可以查看[这篇文章](https://medium.com/flawless-app-stories/gyp-no-xcode-or-clt-version-detected-macos-catalina-anansewaa-38b536389e8d)找到解决的方案。

除了node-gyp，你还需要安装一个叫[bindings](https://www.npmjs.com/package/bindings)的包，这是由于不同版本的node-gyp，抑或是那些没有采用node-gyp进行编译的node扩展，其编译出来的.node文件，可能会放在系统的不同路径之下，如果你在js中引用这个.node文件时，直接hardcode路径，之后如果node-gyp升级后，就可能需要手动修改这个路径，十分麻烦：

```shell
$ npm install bindings
```

而使用bindings后，这个路径的查找过程，bindings就会自动帮你完成，例如你的.node文件叫addon.node，那么只需在js中，直接调用require，就能正确查找到对应的.node文件：

```javascript
const addon = require('bindings')('addon');
```

### Node编程接口

而对于编程接口，现在的node主要提供了三种不同方案：

 1. nan
 2. napi
 3. node-addon-api

其中nan是在node v8.0之前大家使用的最多的方案（node-talib包中的node扩展，就是使用nan与V8进行交互的）。在v8.0之前，由于node官方并没有提供标准的api接口，要写node扩展，只能引用V8的头文件，直接与底层V8引擎交互，不但写起来麻烦，而且随着V8引擎的升级，其暴露的接口也在不断变化，这就导致扩展的开发者需要自己在代码中对不同版本的接口进行兼容处理，十分麻烦。

而nan就是在这个时候诞生的。[nan](https://www.npmjs.com/package/nan)是一个第三方库，其主要就是提供了丰富的宏定义，这些宏定义会自动检查当前V8引擎的版本，并展开成对应的接口调用，这样就可以有效屏蔽因接口变更带来的兼容问题，减少开发者对扩展的维护成本。

虽然nan已经能很好的解决兼容问题，但由于nan的兼容，仅仅是在语言层面的兼容，而在二进制层面，V8引擎的每次升级，接口的调用仍然会发生变化，这使得使用nan所编写的node扩展，还是需要手动升级nan的版本，并重新编译。

不过这种情况在node v8.0后就得到了彻底的解决，因为v8.0后，node官方正式推出自己的api（napi），而且是二进制层面稳定的abi，这样所有已经编译好的扩展，在V8引擎升级的时候，甚至是V8引擎被其他js引擎替代时，都无需重新编译，napi已经帮我们屏蔽了这些差异。

napi是采用c编写，而且相对底层，如果你是使用c++的话，可以采用[node-addon-api](https://www.npmjs.com/package/node-addon-api)，这个包提供了napi的c++类封装，调用起来会比较方便。

而我这次所写的node扩展，正是使用了node-addon-api，可以通过npm来安装：

```shell
$ npm install node-addon-api
```

### 模块初始化

node扩展，本质上是一个node的动态库，在加载到当前node进程时（通过`process.dlopen`方法），需要有一个入口函数，而在node-addon-api中，你可以如下编写这个入口函数：

```cpp
#include <napi.h>

Napi::Object init(Napi::Env env, Napi::Object exports) {
  return exports;
}

NODE_API_MODULE(NODE_GYP_MODULE_NAME, init)
```

其中`<napi.h>`是node-addon-api给我们提供的头文件，而`init`函数，就是我们所需要的入口函数，而且在init函数，我们发现了一个熟悉的身影 -- `exports`对象，没错，这个`exports`对象，就是js模块中的`exports`对象（导出对象），不同的是，这里的`exports`对象是个c++对象（`Napi::Object`）。

### 设置模块属性

既然我们已经找到了`exports`对象，那下一步，就应该是往`exports`对象中，设置我们需要的字段了，要在`Napi::Object`对象中设置属性，可以通过`Set`方法：

```cpp
#include "indicators.h"

// 辅助函数，负责把cpp数组转换成js数组
template<typename T>
Napi::Array toJSArray(Napi::Env env, T const arr[], int len) {
  Napi::Array ret = Napi::Array::New(env);
  for (int i = 0; i < len; i++) {
    ret.Set(i, Napi::Value::From(env, arr[i]));
  }
  return ret;
}

// 入口函数
Napi::Object init(Napi::Env env, Napi::Object exports) {
  // 创建indicators对象，用于存放指标函数信息，其中key为指标函数的名字，如sma
  Napi::Object indicators = Napi::Object::New(env);
  // 从tulip导出所有指标函数
  const ti_indicator_info *ind = ti_indicators;
  while (ind->name != 0) {
    Napi::Object info = Napi::Object::New(env);
    info.Set(Napi::String::New(env, "index"), Napi::Number::New(env, (int)(ind - ti_indicators)));
    info.Set(Napi::String::New(env, "name"), Napi::String::New(env, ind->name));
    info.Set(Napi::String::New(env, "fullName"), Napi::String::New(env, ind->full_name));
    info.Set(Napi::String::New(env, "inputs"), Napi::Number::New(env, ind->inputs));
    info.Set(Napi::String::New(env, "inputNames"), toJSArray(env, ind->input_names, ind->inputs));
    info.Set(Napi::String::New(env, "options"), Napi::Number::New(env, ind->options));
    info.Set(Napi::String::New(env, "optionNames"), toJSArray(env, ind->option_names, ind->options));
    info.Set(Napi::String::New(env, "outputs"), Napi::Number::New(env, ind->outputs));
    info.Set(Napi::String::New(env, "outputNames"), toJSArray(env, ind->output_names, ind->outputs));
    indicators.Set(Napi::String::New(env, ind->name), info);
    ++ind;
  }
  // 把indicators对象写入exports对象中
  exports.Set(Napi::String::New(env, "indicators"), indicators);
  return exports;
}
```

这里我首先创建了一个`indicators`对象，然后从tulip库中（从indicators.h头文件导入），导出所有指标函数信息（包括名称，描述，输入输出，选项等），写入到这个`indicators`对象中，最后再把这个`indicators`对象写入到`exports`对象中，这样，我们就可以在js中访问到这些属性了：

```javascript
const addon = require('bindings')('addon');
// 直接打印属性
console.log(addon.indicators);
```

### tulip库的使用

除了导出属性，我们还需要能调用tulip提供的方法，不过在具体写代码前，我们要先了解一下，如何在cpp中调用tulip提供的方法。而正如我在上文中所说的那样，tulip的方法调用非常的简单，从tulip提供的文档可以看到，所有tulip提供的指标函数，其调用方法都是一致的，tulip为每个指标函数提供了两个方法：

```c
int ti_XXX_start(TI_REAL const *options);
int ti_XXX(int size,
             TI_REAL const *const *inputs,
             TI_REAL const *options,
             TI_REAL *const *outputs);
```

`XXX`就是具体的指标名称，例如`ti_sma_start`和`ti_sma`就是关于简单移动平均线的计算。其中的`ti_XXX_start`方法，是用于计算当前选项下，输入数组的长度与输出数组长度的差值，在初始化输入输出参数时会使用到。而`ti_XXX`方法则是执行具体的指标计算。而在tulip文档中，还给我们提供了一个例程：

```c
// 准备好需要进行计算的数据，例如某个股票的最近10天收盘价，放入输入数组
const double data_in[] = {5,8,12,11,9,8,7,10,11,13};
const int input_length = sizeof(data_in) / sizeof(double);

// 设置选项，这里3代表的是sma的计算周期为3天
const double options[] = {3};

// 找到选项下，输入数组和输出数组的差值
const int start = ti_sma_start(options);
// 根据上面得出的差值，计算出输出数组的长度
const int output_length = input_length - start;
// 根据计算到的输出数组长度分配内存空间，tulip会把计算结果写入这些内存中
double *data_out = malloc(output_length * sizeof(double));

// 执行指标计算，具体到这里就是计算过去十天的sma
// 这里要注意的是tulip的输入输出都采用二维数组
// 这样对于有多个输入或输出的指标，就可以保持函数签名一致
const double *all_inputs[] = {data_in};
double *all_outputs[] = {data_out};
int error = ti_sma(input_length, all_inputs, options, all_outputs);

// tulip中所有指标计算，都会返回状态码
assert(error == TI_OKAY);
```

可以看出来，tulip的方法调用十分简单，之后，我们会在js中重新实现上面这个例程。

### 设置模块方法

在知道如何使用tulip进行指标计算后，我们就可以在node扩展中实现对应的方法了：

```cpp
// 辅助函数，负责把js数组转换成cpp数组
double *makeArray(Napi::Array arr) {
  const int size = arr.Length();
  double *ret = (double *)malloc(sizeof(double) * size);
  for (int i = 0; i < size; i++) {
    ret[i] = arr.Get(i).As<Napi::Number>().DoubleValue();
  }
  return ret;
}

Napi::Value callByIndex(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

#define ASSERT(cond, error)\
  if (!(cond)) { Napi::TypeError::New(env, error).ThrowAsJavaScriptException(); return env.Null(); }

  // 参数校验
  ASSERT(info.Length() >= 3, "Wrong number of arguments. Expecting: index, inputs, and options.")
  ASSERT(info[0].IsNumber(), "Expecting first argument to be function index number.")
  ASSERT(info[1].IsArray(), "Expecting second argument to be array of inputs.")
  ASSERT(info[2].IsArray(), "Expecting third argument to be array of options.")

  // 根据传入的index，查找指标信息对象
  const int index = info[0].As<Napi::Number>().Int32Value();
  ASSERT(index >= 0 || index < TI_INDICATOR_COUNT, "Invalid indicator index.")
  const ti_indicator_info *ind = ti_indicators + index;

  // 把传入的inputs参数，转换为tulip使用的输入数组
  Napi::Array inputs = info[1].As<Napi::Array>();
  ASSERT((unsigned int)ind->inputs == inputs.Length(), "Invalid number of inputs.")
  double *input_arr[TI_MAXINDPARAMS] = {0};
  int input_size = -1;
  for (int i = 0; i < ind->inputs; i++) {
    ASSERT(inputs.Get(i).IsArray(), "Expecting second argument to be array of arrays of numbers.")
    Napi::Array input = inputs.Get(i).As<Napi::Array>();
    int size = input.Length();
    if (input_size < 0 || size < input_size) {
      input_size = size;
    }
    input_arr[i] = makeArray(input);
  }

  // 把传入的options参数，转换为tulip使用的选项数组
  Napi::Array options = info[2].As<Napi::Array>();
  ASSERT((unsigned int)ind->options == options.Length(), "Invalid number of options.")
  double *option_arr = makeArray(options);

  // 调用start方法，计算输入数组和输出数组的长度差
  // 并据此计算出输出数组的长度，并为输出数组分配内存
  const int start = ind->start(option_arr);
  const int output_size = start >= 0 ? input_size - start : 0;
  double *output_arr[TI_MAXINDPARAMS] = {0};
  for (int i = 0; i < ind->outputs; i++) {
    output_arr[i] = (double *)malloc(sizeof(double) * output_size);
  }

  // 进行指标计算
  int result = ind->indicator(input_size, input_arr, option_arr, output_arr);
  Napi::Value ret;
  // 如果计算成功，则把输出数组转换成输出的js对象，否则返回null
  if (result == TI_OKAY) {
    Napi::Array outputs = Napi::Array::New(env);
    for (int i = 0; i < ind->outputs; i++) {
      outputs.Set(i, toJSArray(env, output_arr[i], output_size));
    }
    ret = outputs;
  } else {
    ret = env.Null();
  }

  // 释放之前分配的内存
  for (int i = 0; i < ind->inputs; i++) {
    free(input_arr[i]);
  }
  for (int i = 0; i < ind->outputs; i++) {
    free(output_arr[i]);
  }
  free(option_arr);

#undef ASSERT

  return ret;
}
```

这个方法有点长，但主要就是做了两个事情，第一个是参数校验，由于js是弱类型的，任何从js中传入的参数，在正式参与计算前，都需要做类型判断，并转换成对应的cpp数据类型。这里我写了一个宏来减少代码量，大家可以自行展开。

在完成参数校验和转换后，接下来第二个就是调用tulip提供的方法了，实际的调用和上一节中说到的大同小异，具体大家可以看我上面的注释。最后就是把计算结果返回给js。

接下来，就把这个方法写入`exports`对象吧：

```cpp
Napi::Object init(Napi::Env env, Napi::Object exports) {
  Napi::Object indicators = Napi::Object::New(env);
  ...
  exports.Set(Napi::String::New(env, "callByIndex"), Napi::Function::New(env, callByIndex));
  return exports;
}
```

至此，我们已经完成整个node扩展的代码编写。

### 编译node扩展

是时候编译我们的node扩展了，首先我们要为node-gyp准备一个binding.gyp配置文件，这是一个类json文件：

```json
{
  "targets": [
    {
      "target_name": "addon",
      "cflags!": [ "-fno-exceptions" ],
      "cflags_cc!": [ "-fno-exceptions" ],
      "sources": [
        "indicators.cc",
        "indicators/tiamalgamation.c"
      ],
      "include_dirs": [
        "<!@(node -p \"require('node-addon-api').include\")",
        "indicators"
      ],
      "defines": [ "NAPI_DISABLE_CPP_EXCEPTIONS" ],
    }
  ]
}
```

其中，`target_name`就是扩展输出的名字（包括.node文件的名字），`sources`就是需要编译的c/c++文件，这里除了要引用我们编写的cpp文件，还需要把tulip的c文件也引用进来。

同时在package.json文件中，添加一个gypfile字段，以及build命令：

```json
{
  "scripts": {
    "build": "node-gyp rebuild"
  },
  "gypfile": true
}
```

最后在命令行中执行：

```shell
$ npm run build
> node-gyp rebuild

  CXX(target) Release/obj.target/addon/indicators.o
  CC(target) Release/obj.target/addon/indicators/tiamalgamation.o
  SOLINK_MODULE(target) Release/addon.node
```

这样，我们的node扩展就编译完成了。

### 从js中调用node扩展

这时我们的node扩展已经是可用状态了，不过为了在js中使用方便，我们还需要有一个js的胶水文件，用于对外暴露我们的接口：

```javascript
// tupli.js文件
const addon = require('bindings')('addon');

exports.indicators = addon.indicators;
exports.callByIndex = addon.callByIndex;

for (const [name, indicator] of Object.entries(addon.indicators)) {
  exports[name] = (...args) => {
    const inputs = args.slice(0, indicator.inputs);
    const options = [];

    const [rest = {}] = args.slice(indicator.inputs);
    for (const name of indicator.optionNames) {
      options.push(rest[name] || 0);
    }

    const outputs = addon.callByIndex(indicator.index, inputs, options);
    if (!outputs) {
      return null;
    }

    const result = {}
    for (const name of indicator.outputNames) {
      result[name] = outputs.shift();
    }
    return result;
  }
}
```

这个文件就不在这里展开了，主要就是封装一下接口，调整好输入输出参数等，接下来，我们就可以在js代码中愉快的和我们自己编写的node扩展交互了，这里我还是使用了上面计算sma的例程：

```javascript
const indicators = require('../tulip');
// 最近10天股票的收盘价
const close = [5, 8, 12, 11, 9, 8, 7, 10, 11, 13];
// 设置sma的计算周期为3天
const options = {
  period: 3,
};
// 输出结果
const { sma } = indicators.sma(close, options);
console.log(sma);
```

以上！！！
