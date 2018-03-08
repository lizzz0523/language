
# ctypes使用指南

## 加载动态链接库

通过ctypes中的cdll对象（在windows系统上还有windll和oledll对象），我们可以导入系统中的动态链接库。通过访问这些对象的属性，我们可以加载不同的动态链接库。cdll对象导入的库，遵循cdecl调用方式，而windll对象导入的库，则遵循stdcall调用方式，oledll对象导入的库同样遵从stdcall调用方式，但和普通的stdcall调用方式不同，oledll会假设方法调用会返回一个windows的HRESULT错误码，python会利用该错误码自动的抛出错误。

以下是在windows系统中调用ctypes的一些例子，值得注意的是，msvcrt是ms的标准c库，其中包含了大部分标准的c函数，这些函数都遵循cdecl调用方式。


```python
from ctypes import *
```


```python
print(windll.kernel32)
```

    <WinDLL 'kernel32', handle 770f0000 at 0x2b4c4a8>
    


```python
print(cdll.msvcrt)
```

    <CDLL 'msvcrt', handle 7fefd8b0000 at 0x4a539b0>
    


```python
print(cdll.wpcap)
```

    <CDLL 'wpcap', handle 180000000 at 0x4a53c88>
    


```python
libc = cdll.msvcrt
```

在windows系统下，载入动态链接库时，ctypes会自动追加上.dll后缀。也就是说`cdll.msvcrt`实际指的的`msvcrt.dll`库

而在linux系统上，则需要手动的通过指定文件路径（名字+后缀）的方式加载对应的动态链接库，而不能简单的通过访问对象属性的方式来加载。除了可以通过调用`cdll.LoadLibrary`方法以外，还可以通过调用CDLL构造函数，创建cdll实例的方式来加载动态链接库。


```python
# for linux
# cdll.LoadLibrary('libc.so.6')
```


```python
# for linux
# libc = CDLL('libc.so.6')
# libc
```

## 访问动态链接库中的函数

通过访问动态链接库对象的属性，我们可以获取到库中包含的函数


```python
libc.printf
```




    <_FuncPtr object at 0x00000000048E9C78>




```python
windll.kernel32.GetModuleHandleA
```




    <_FuncPtr object at 0x00000000048E9D48>




```python
# windll.kernel32.MyOwnFunction
```

值得注意的是，像kernel32和user32这样的win32系统动态库，通常都会导入ansi和unicode两个版本的函数。unicode版本的函数会在函数名末尾追加W后缀，而ansi版本的函数则会在函数名末尾追加A后缀。例如win32中的GetModuleHandle函数（通过模块名称返回模块句柄）,其c语言中的函数原型如下，这里使用一个宏定义，根据系统中系否支持unicode，来导出两个函数的其中一个：


```python
# /* ANSI version */
# HMODULE GetModuleHandleA(LPCSTR lpModuleName);
# /* UNICODE version */
# HMODULE GetModuleHandleW(LPWSTR lpModuleName);
```

windll并不会隐式的选择两个函数的其中一个，而是需要你明确通过不同的名称来选择不同版本的函数，同时要求在调用相应的函数时，传入数据格式正确的参数。

某些时候，有的动态库，会导入一些函数，其名称不符合python的标识符规范（例如"??2@YAPAXI@Z"），这时候，你可以使用`getattr`方法来获取这些函数:


```python
# getattr(cdll.msvcrt, "??2@YAPAXI@Z")
```

在windows系统上，有的动态库并不是使用名称而是序数的方式来导出函数，这时可以通过数组下标的方式，访问这些函数：


```python
cdll.kernel32[1]
```




    <_FuncPtr object at 0x00000000048E9E18>




```python
# cdll.kernel32[0]
```

## 调用动态链接库中的函数

你可以像调用python中其他函数一样调用动态库中的函数。下面是一个调用`time`函数（返回系统时间戳），和调用`GetModuleHandleA`函数（返回模块句柄）的例子。

调用这些函数，我们需要传入一个`Null`指针（在这里的`None`）：


```python
libc.time(None)
```




    1520477581




```python
hex(windll.kernel32.GetModuleHandleA(None))
```




    '0x1c030000'



在windows系统下，当你传入错误数量的参数或者以错误的方式调用该函数，ctypes会尝试继续调用该函数，并且通过抛出错误来告诉调用者：


```python
# windll.kernel32.GetModuleHandleA()
```


```python
windll.kernel32.GetModuleHandleA(0, 0)
```




    469958656



如果以cdecl调用方式调用stdcall函数，同样也会抛出错误，反之亦然：


```python
cdll.kernel32.GetModuleHandleA(None)
```




    469958656




```python
windll.msvcrt.printf("spam")
```




    1



为了使用正确的方式调用函数，你需要查阅相关的c头文件或文档。

在windows系统上，当使用错误的参数调用函数时，ctypes通过win32结构化异常处理来避免python因为通用保护错误而崩溃。


```python
# windll.kernel32.GetModuleHandleA(32)
```

虽然如此，ctypes在许多情况下，仍然会引python崩溃，所以还是要小心。

`None`，`int`，`long`，`str`和`unicode`是为数不多的几种，能直接作为参数传入动态库函数的python原生数据类型。其中`None`对应c中的`Null`指针，`str`和`unicode`会转化为指针，指向其值所在的数据块地址（对应c中的`char *`和`wchar_t *`），而`int`型和`long`型则转化为c中的`int`型。

在继续讨论使用其他参数类型调用动态库函数前，我们先来学习一下ctypes中其他数据类型。

## 基础数据类型

ctypes中定义了一系列与c兼容的基础数据类型：

|ctypes type|C type|Python type|
|-|-|-|
|c_char|char|1-character string|
|c_wchar|wchar_t|1-character unicode string|
|c_byte|char|int/long|
|c_ubyte|unsigned char|int/long|
|c_short|short|int/long|
|c_ushort|unsigned short|int/long|
|c_int|int|int/long|
|c_uint|unsigned int|int/long|
|c_long|long|int/long|
|c_ulong|unsigned long|int/long|
|c_longlong|int64 or long long|int/long|
|c_ulonglong|unsigned int64 or unsigned long long|int/long|
|c_float|float|float|
|c_double|double|float|
|c_char_p|char * |string or None|
|c_wchar_p|wchar_t * |unicode or None|
|c_void_p|void * |int/long or None|

这些数据类型，都可以通过调用对应的函数，并传入一个可选的初始值来创建：


```python
c_int()
```




    c_long(0)




```python
c_char_p(b"Hello World")
```




    c_char_p(77772032)




```python
c_ushort(-3)
```




    c_ushort(65533)



这些数据类型都是可变的，可以在声明以后，修改其值：


```python
i = c_int(42)
```


```python
print(i)
```

    c_long(42)
    


```python
i.value = -99
```


```python
print(i)
```

    c_long(-99)
    

对于`c_char_p`，`c_wchar_p`和`c_void_p`这样的指针类型，重新赋值实际上是改变指针的指向，而并不会修改其内存中的数据：


```python
s = b"Hello World"
```


```python
c_s = c_char_p(s)
```


```python
print(c_s)
```

    c_char_p(77773232)
    


```python
c_s.value = b"Hi World"
```


```python
print(c_s)
```

    c_char_p(77772464)
    


```python
print(s)
```

    b'Hello World'
    

然而，这里你需要特别小心，不要把这些指针参数传递给需要修改内存中数据的函数，如果你要调用这些需要修改内存数据的函数，请使用ctypes提供的`create_string_buffer`方法来创建这些参数。

## 再谈函数调用


```python
printf = libc.printf
```


```python
printf("Hello %s\n", b"World!")
```




    1




```python
printf("Hello %S", "World!")
```




    1




```python
printf("%d bottles of beer\n", 42)
```




    0




```python
# printf("%f bootles of beer\n", 42.5)
```

正如之前所说的，在python中，除了None，int，long，str，unicode以外，其他的数据类型都需要先包装到相应的ctypes类型中，然后他们才能以正确的形式传入到c函数中：


```python
printf("An int %d, a double %f\n", 1234, c_double(3.14))
```




    1



## 使用自定义的数据类型作为参数

你也可以通过定制类型转换，使你自定义的类实例能作为参数传入c函数中。ctypes内部会通过查找类实例中的`_as_parameter_`属性来作为该实例的值传入c函数中。当然`_as_parameter_`属性的类型也必须是`int`，`long`，`str`，`unicode`之一，或者是ctypes中的数据类型：


```python
class Bottles(object):
    def __init__(self, number):
        self._as_parameter_ = c_int(number)
        
bottles = Bottles(42)
printf("%d bottles of beer\n", bottles)
```




    0



如果你不希望把实例的数据存储在`_as_parameter_`属性上，你也可以通过实现一个特定的实例方法来返回你的数据。

# 指定c函数的参数类型（函数原型）

我们可以通过特定的`argtypes`属性来指定那些从动态库导出的c函数的参数类型。

`argtypes`属性必须是一个c数据类型的数组（`printf`函数在这里可能不是一个特别好的例子，这是因为他的参数数量和参数类型的会因为不同的format string而改变，在这种情况下，很难把`argtypes`属性的作用所明白）:


```python
printf.argtypes = [c_wchar_p, c_wchar_p, c_int, c_double]
printf("String %s, Int, %d, Double %f\n", "Hi", 10, 2.2)
```




    1



通过`argtypes`属性指定参数类型，能起到类型校验的作用（就像c中的函数原型），同时ctypes会使用这些指定的数据类型来尝试对参数进行类型转换：


```python
# printf("%d %d %d", 1, 2, 3)
```

如果你需要向c函数传入自定义的类实例，那么你可以在这些类上实现一个名叫`from_param`的类实例方法，同时把类的构造函数放入该c函数的`argtypes`类型序列中。当你把该类实例传入c函数时，类实例的`from_param`方法就会被调用。这时你可以在`from_param`方法中实现类型校验或者任何其他操作以保证该类实例的值是正确的。最后`from_param`方法返回实例本身，或者其`_as_parameter_`属性，或者任何你想传入c函数的值。再一次说明，该值只能是`int`，`long`，`str`，`unicode`或者ctypes数据类型，或者其他包含正确`_as_parameter_`属性的对象。

# 返回值类型

在默认的情况下，c函数都会假设以`int`作为返回值的类型。你也可以通过c函数的`restype`属性来指定特定的特定的返回值类型。

这里是一个更加具体的例子，在这里例子中，我们调用了`strchr`函数，该函数接受一个字符串指针和一个字符，并且返回另一个字符串指针：


```python
strchr = libc.strchr
```


```python
strchr(b"abcdef", ord("d"))
```




    77839971




```python
strchr.restype = c_char_p
```


```python
strchr(b"abcdef", ord("d"))
```




    b'def'




```python
strchr(b"abcdef", ord("x"))
```

如果你希望避免像上面一样重复调用`ord("x")`来把字符串`"x"`转换成字符类型，你可以通过设定特定的`argtypes`属性，来使函数的第二个参数自动转换成一个字符类型：


```python
strchr.restype = c_char_p
```


```python
strchr.argtypes = [c_char_p, c_char]
```


```python
strchr(b"abcdef", b"d")
```




    b'def'




```python
# strchr(b"abcdef", b"def")
```


```python
strchr(b"abcdef", b"x")
```

你也可以把一个函数或者类构造函数作赋值给c函数的restype属性。如果这个c函数返回了一个整型数，那么这个整型数就会作为参数传入你指定的函数或者类构造函数中，其返回值会代替原来的整型数作为c函数的返回值。这对于那些需要检测c函数返回值，并在错误时抛出异常的情况非常有用：


```python
GetModuleHandle = windll.kernel32.GetModuleHandleA
```


```python
def ValidHandle(value):
    if value == 0:
        raise WinError()
    return value
```


```python
GetModuleHandle.restype = ValidHandle
```


```python
GetModuleHandle(None)
```




    469958656




```python
# GetModuleHandle("something silly")
```

上面的`WinError`函数会调用windows下的`FormatMessage`接口来获得错误码对应的错误描述，并且返回一个异常。`WinError`接受一个可选的错误码作为参数，如果参数为空，则会调用`GetLastError`接口来获取对应的错误。

如果你需要更加强大的错误校验机制，可以尝试一下`errcheck`属性。具体请查阅相关的文档。

## 传递指针（引用调用）

有时候，出于需要修改指定内存地址上的数据，或者避免因按值调用对于复制大块数据带来的性能问题等原因，某些c函数需要传入一个指针作为参数，也就是我们常说的按引用调用。

ctypes给我们提供了一个`byref`函数以实现这种按引用调用，除此以外我们还可以使用ctypes中的`pointer`函数来达到相同的目的，但相比与`byref`函数，`pointer`函数需要完成更多的工作来创建一个真实的指针对象，这使得在你并不需要这个指针对象的情况下，`byref`是一个更好的选择：


```python
i = c_int()
```


```python
f = c_float()
```


```python
s = create_string_buffer(b'\000' * 32)
```


```python
print(i.value, f.value, repr(s.value))
```

    0 0.0 b''
    


```python
libc.sscanf(b"1 3.14 Hello", b"%d %f %s", byref(i), byref(f), s)
```




    3




```python
print(i.value, f.value, repr(s.value))
```

    1 3.140000104904175 b'Hello'
    

## 结构体和联合体

如果需要在python中定义c中的结构体或者联合体，则需要创建相应的类，并继承ctypes中的`Structure`和`Union`基类。在这些子类中，需要包含一个`_fields_`属性，其值为一个包含一系列二元组的数组，每个二元组中则包含结构体或者联合体中的字段名称和字段类型。

其中的字段类型，必须是一个ctypes的数据类型，例如像`c_int`或者其他ctypes中的合法类型，包括`structure`，`union`，`pointer`等。

这里是一个关于POINT结构体的简单例子，POINT结构体中包含`x`，`y`两个整型字段，在例子中，我们展示如何初始化一个结构体：


```python
class POINT(Structure):
    _fields_ = [("x", c_int),
                ("y", c_int)]
```


```python
point = POINT(10, 20)
```


```python
print(point.x, point.y)
```

    10 20
    


```python
point = POINT(y=5)
```


```python
print(point.x, point.y)
```

    0 5
    


```python
# POINT(1, 2, 3)
```

你可以根据自身的需求，创建更加复杂的结构体。结构体除了包含基础类型外，也可以包含其他的结构体作为字段类型。

下面是一个名叫RECT的复合结构体，其中包括了两个分别名为`upperleft`，`lowerright`的POINT类型数据：


```python
class RECT(Structure):
    _fields_ = [("upperleft", POINT),
                ("lowerright", POINT)]
```


```python
rc = RECT(point)
```


```python
print(rc.upperleft.x, rc.upperleft.y)
```

    0 5
    


```python
print(rc.lowerright.x, rc.lowerright.y)
```

    0 0
    

ctypes提供了多种方式用于初始化复合结构体：


```python
rc = RECT(POINT(1, 2), POINT(3, 4))
```


```python
rc = RECT((1, 2), (3, 4))
```

通过构造函数本身可以获取到不同字段的信息，这在开发调试阶段是相对有用的：


```python
print(POINT.x)
```

    <Field type=c_long, ofs=0, size=4>
    


```python
print(POINT.y)
```

    <Field type=c_long, ofs=4, size=4>
    

## 结构体和联合体中的对齐和字节顺序

默认情况下，`Structure`和`Union`中采用的是与c编译器相同的对齐方式，但你可以通过在子类中设置`_pack_`属性来修改默认的行为。`_pack_`属性必须是一个正整数，用于指定字段的最大对齐方式，类似MSVC中的`#pragma pack(n)`。

`Structure`和`Union`内部采用的是机器本地的字节顺。如果你想构建一个非本地字节顺序的结构体，你可以尝试使用`BigEndianStructure`，`LittleEndianStructure`，`BigEndianUnion`，`LittleEndianUnion`中的其中之一作为基类。但这些基类中不能包含指针类型的字段。

## 结构体和联合体中的位字段

通过ctypes，我们可以创建带有位字段的结构体和联合体。位字段只能是一个整型字段，通过在`_fields_`属性上对应字段的二元组上加入第三项来指定位字段的位宽：


```python
class Int(Structure):
    _fields_ = [("first_16", c_int, 16),
                ("second_16", c_int, 16)]
```


```python
print(Int.first_16)
```

    <Field type=c_long, ofs=0:0, bits=16>
    


```python
print(Int.second_16)
```

    <Field type=c_long, ofs=0:16, bits=16>
    

## 数组

数组是包含固定数量的相同类型数量的序列。

要在定义ctypes类型的数组，最好的方式是采用一个正整数乘以一个ctypes数据类型：


```python
TenPointsArrayType = POINT * 10
```

下面的例子中，我们构造了一个特殊的结构体，里面包含了一个包含四个`POINT`的数组字段和其他的字段：


```python
class POINT(Structure):
    _fields_ = [("x", c_int), ("y", c_int)]
```


```python
class MyStruct(Structure):
    _fields_ = [("a", c_int),
                ("b", c_float),
                ("point_array", POINT * 4)]
```


```python
print(len(MyStruct().point_array))
```

    4
    

创建一个数组实例和创建其他数据类型实例一样，通过调用构造函数即可：


```python
arr = TenPointsArrayType()
```


```python
for pt in arr:
    print(pt.x, pt.y)
```

    0 0
    0 0
    0 0
    0 0
    0 0
    0 0
    0 0
    0 0
    0 0
    0 0
    

上面的这段代码会打印出一些列的0，这是因为数组的内容被默认初始化为0了。

我们也可以通过传入正确的数据类型来进行数组的初始化：


```python
TenIntergers = c_int * 10
```


```python
ii = TenIntergers(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```


```python
print(ii)
```

    <__main__.c_long_Array_10 object at 0x0000000004A28E48>
    


```python
for i in ii:
    print(i)
```

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    

## 指针

通过调用`pointer`函数并传入一个ctypes数据类型实例，我们可以创建一个指针实例：


```python
i = c_int(42)
```


```python
pi = pointer(i)
```


```python
pi
```




    <ctypes.wintypes.LP_c_long at 0x4a28cc8>



指针实例包含一个`contents`属性，通过这个属性，我们可以获取指针所指向的内容，也就是上面的变量`i`:


```python
pi.contents
```




    c_long(42)



但需要注意的是，ctypes并没有真的把原数据实例返回给你，而是在每一次你读取`contents`属性时，临时创建了一个新的数据实例，其中包含你的原始数据：


```python
pi.contents is i
```




    False




```python
pi.contents is pi.contents
```




    False



对指针实例的`contents`属性赋值，会修改该指针的指向：


```python
i = c_int(99)
```


```python
pi.contents = i
```


```python
pi.contents
```




    c_long(99)



我们也可以通过下标访问指针实例所指向内存中的数据：


```python
pi[0]
```




    99



通过下标来对指针实例赋值，会修改指针所指向内存中的数据：


```python
print(i)
```

    c_long(99)
    


```python
pi[0] = 22
```


```python
print(i)
```

    c_long(22)
    

当然我们也可以通过非0下标来访问指针实例所指向的数据，但前提是你知道你在做什么，这和在c语言里操作指针是一样的，你可以通过指针获取和改变任意内存中的数据。通常来说，只有当你从一个c函数中获取到一个指针，并且明确知道这个指针指向的是一个数组时而不是一个单值时，你才需要用到这种操作。

其实`pointer`函数在底层并不仅仅是简单的创建一个指针实例，在创建这个实例之前，他首先会创建一个相应的指针类型。通过调用`POINTER`函数，并传入任意的ctypes数据类型，`POINTER`函数就会创建相应的指针类型并返回：


```python
PI = POINTER(c_int)
```


```python
PI
```




    ctypes.wintypes.LP_c_long




```python
# PI(42)
```


```python
PI(c_int(42))
```




    <ctypes.wintypes.LP_c_long at 0x4ae6048>



如果在创建一个指针实例时不传入任何参数，那么ctypes就会创建一个空指针，如果转换成布尔值则等于`False`：


```python
null_ptr = POINTER(c_int)()
```


```python
print(null_ptr)
```

    <ctypes.wintypes.LP_c_long object at 0x0000000004AE60C8>
    


```python
print(bool(null_ptr))
```

    False
    

在获取空指针的内容时，ctypes会进行检测并抛出异常：


```python
# null_ptr[0]
```


```python
# null_ptr[0] = 4
```


```python
null_ptr.contents = c_int(4)
```

## 类型转换

通常情况下，ctypes会进行严格的类型校验。也就是说，如果你在c函数的`argtypes`属性中包含一个`POINTER(c_int)`类型，或者作为一个结构体的某个字段类型，那么这些地方就只使用`POINTER(c_int)`类型的变量。不过也有例外的情况，在这些情况下，ctypes能接受其他类型的数据，例如在这个例子中传入了一个整型数组代替`POINTER(c_int)`指针实例：


```python
class Bar(Structure):
    _fields_ = [("count", c_int), ("values", POINTER(c_int))]
```


```python
bar = Bar()
```


```python
bar.values = (c_int * 3)(1, 2, 3)
```


```python
bar.count = 3
```


```python
for i in range(bar.count):
    print(bar.values[i])
```

    1
    2
    3
    

如果你需要把指针实例设置成空指针，只要赋值成`None`即可：


```python
bar.values = None
```

在某些情况下，你需要处理的数据类型并不兼容，在c语言中，你可以通过类型转换把一种数据类型转换成另外一种数据类型。而ctypes也提供了一个`cast`函数完成相同的功能。在上面的`Bar`结构体定义中，接受一个`POINTER(c_int)`指针类型或者一个`c_int`数组类型的数据作为其`values`字段的值。对于其他类型的数据，则会抛出异常：


```python
# bar.values = (c_byte * 4)()
```

在这种情况下，我们就可以使用`cast`函数。

`cast`函数能把某种ctypes数据类型指针转换成另一种ctypes数据类型指针。`cast`函数接受两个参数，第一个参数是某个需要转换的ctypes类型数据指针或者任何可以与指针等价的对象实例（例如数组），第二个参数需要转换的需要转换指针类型。`cast`函数会返回转换后的指针实例，并且转换前后的两个指针实例指向同一块内存：


```python
a = (c_byte * 4)()
```


```python
cast(a, POINTER(c_int))
```




    <ctypes.wintypes.LP_c_long at 0x4ae6348>



因此利用`cast`函数，我们就可以把转换后的指针实例赋值给`Bar`结构体的`values`字段：


```python
bar = Bar()
```


```python
bar.values = cast((c_byte * 4)(), POINTER(c_int))
```


```python
print(bar.values)
```

    <ctypes.wintypes.LP_c_long object at 0x0000000004AE6248>
    

## 不完整类型

不完整类型，是指某些字段的数据类型没有给出完整定义的结构体，联合体或数组。在c语言中，这些不完整字段的类型只有前向声明，其实际定义要在后面再另外给出：


```python
# struct cell;
# struct {
#     char *name;
#     struct cell *next;
# }
```

如果我们直接把上面的c代码转换成ctypes定义，就会得到如下结果，但实际上这样的ctypes定义并不能正常工作：


```python
# class cell(Structure):
#     _fields_ = [("name", c_char_p),
#                 ("next", POINTER(cell))]
```

这是因为这里的`cell`类型，在声明`cell`类时，还没有生效。在ctypes中，我们可以通过先定义`cell`类，然后再设置其`_fields_`属性的方式完成不完整类型的定义：


```python
class cell(Structure):
    pass
```


```python
cell._fields_ = [("name", c_char_p),
                 ("next", POINTER(cell))]
```

接下来，我们将尝试创建两个`cell`实例。并且让其中一个指向另一个，最终实现一条指针链：


```python
c1 = cell()
```


```python
c1.name = b"foo"
```


```python
c2 = cell()
```


```python
c2.name = b"bar"
```


```python
c2.next = pointer(c1)
```


```python
c1.next = pointer(c2)
```


```python
p = c1
```


```python
for i in range(8):
    print(p.name)
    p = p.next[0]
```

    b'foo'
    b'bar'
    b'foo'
    b'bar'
    b'foo'
    b'bar'
    b'foo'
    b'bar'
    

## 回调函数

在ctypes中，运行我们创建一个函数指针指向python中的函数，我们通常称这些函数为回调函数。

首先，你需要为这些回调函数创建一个ctypes类，在这个类中需要指定函数的调用方式，返回值类型，以及参数的数量和类型。

我们可以利用`CFUNCTYPE`工厂函数创建一个遵循`cdecl`调用方式的函数指针类，如果是在windows系统上，我们也可以利用`WINFUNCTYPE`工厂函数创建一个遵循`stdcall`调用方式的函数指针类。

无论使用那种方法，我们都需要以回调函数的返回值类型作为第一个参数传入工厂函数，而工厂函数的其余参数，则为回调函数所需的参数类型。

接下来，我会以c库中的`qsort`函数作为例子说明如果在ctypes中定义函数指针类型。`qsort`函数会接受一个回调函数用于辅助对一个整型数组进行排序：


```python
IntArray5 = c_int * 5
```


```python
ia = IntArray5(5, 1, 7, 33, 99)
```


```python
qsort = libc.qsort
```


```python
qsort.restype = None
```

要调用`qsort`函数，我们需要传入一个待排序数组的头指针，数组的长度，数组中数据类型的字宽，以及前面所说的回调函数的指针。在排序的过程中，`qsort`内部会调用回调函数，并传入两个待排序的整型数指针，如果第一个整数比第二个小，则回调函数需要返回一个负数，如果第一个整数比第二个大，则返回一个正数，如果两个整数相等，返回0即可。

因此，我们这里定义的函数指针类型需要接受两个整型数指针，并且返回一个整数：


```python
CMPFUNC = CFUNCTYPE(c_int, POINTER(c_int), POINTER(c_int))
```

首先对于这个回调函数的第一个版本，我们只是简单的打印一下需要比较的两个整数，并且返回0：


```python
def py_cmp_func(a, b):
    print("py_cmp_func", a, b)
    return 0
```

接下来，我们把这个python中定义的回调函数，转换成一个ctypes类型的回调函数指针：


```python
cmp_func = CMPFUNC(py_cmp_func)
```

一切准备就绪：


```python
qsort(ia, len(ia), sizeof(c_int), cmp_func)
```

    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    py_cmp_func <ctypes.wintypes.LP_c_long object at 0x0000000004AE6F48> <ctypes.wintypes.LP_c_long object at 0x0000000004AE9048>
    

上面我们介绍ctypes指针是，曾说过如何获取到指针所指内存中的数据，接下来，我们写出第二版的回调函数：


```python
def py_cmp_func(a, b):
    print("py_cmp_func", a[0], b[0])
    return 0
```


```python
cmp_func = CMPFUNC(py_cmp_func)
```

下面是我们在windows系统上执行`qsort`得到的结果：


```python
qsort(ia, len(ia), sizeof(c_int), cmp_func)
```

    py_cmp_func 7 1
    py_cmp_func 33 1
    py_cmp_func 99 1
    py_cmp_func 5 1
    py_cmp_func 7 5
    py_cmp_func 33 5
    py_cmp_func 99 5
    py_cmp_func 7 99
    py_cmp_func 33 99
    py_cmp_func 7 33
    

有趣的是，如果你在linux系统上执行同样的代码，你会发现linux系统上回调函数被调用的次数比windows系统上少，貌似linux系统的'qsort'的性能更好一些。

好了，到了最后一个版本的回调函数了，在这个版本中，我们真正的对比两个整数的大小，并返回有用的结果：


```python
def py_cmp_func(a, b):
    return a[0] - b[0]
```


```python
cmp_func = CMPFUNC(py_cmp_func)
```


```python
qsort(ia, len(ia), sizeof(c_int), cmp_func)
```


```python
for i in ia:
    print(i)
```

    1
    5
    7
    33
    99
    

最后需要注意的是，在c函数调用期间，你需要保证你的回调函数一直处于内存中而不会被以外的垃圾回收了，ctypes本身并不会帮你完成这个任务。一旦你的回调函数被以外回收，那么你的程序就会崩溃。
