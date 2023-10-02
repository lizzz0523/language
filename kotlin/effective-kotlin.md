# 第一部份：什么才是好的代码

## 好的代码是安全的代码

### 限制可变性

可变性的危害：
1. 可变的对象，难以追踪。引起变化的因素众多，关系错中复杂
2. 可变的对象，难以解读。这一刻确定的值，下一刻就会发生变化
3. 可变的对象，在多线程环境下需要同步机制，限制并发冲突
4. 可变的对象，难以测试，状态太多，很难考虑完全
5. 可变的对象，有时候需要实现变更通知，以达到状态同步

>说白了就是容易出bug，明明现在看起来正确的代码，因为他包含可变的对象，下一刻就会变成有bug的状态，而且你还不好定位这个bug是从哪里来的。特别是在多线程的环境下，就更加麻烦了。
>
>虽然可变性是可恶的，但他就是方便，性能好，程序没可变性还真不行，关键是要限制可变性。只在有限的地方使用。这样出bug了，好定位

不可变的好处：
1. 不可变的对象，容易追逐，因为他是不可变的，基本就是不用追踪
2. 不可变的对象，在多线程的时候，无需解决并发冲突，因为没有冲突
3. 不可变的对象，可以用于缓存的场景，因为你总是期望从缓存里取出来的对象，就是之前存进去的对象，可变了就不符合预期了
4. 不可变的对象，你无需做额外的防御性措施，因为他就是不可变的，在你交出去这个对象时，就不需要copy一次，避免自身被外部修改。而且即使在需要copy的地方，就做个浅拷贝即可
5. 不可变的对象，适用范围更广，可以用于组成其他的不可变对象或者可变对象，但是可变的对象，就只能永远可变
6. 不可变的对象，可以作为set的值，或者map的key，因为很多时候这些地方都需要对对象本身进行一次hashCode计算，如果是可变对象，前后的hashCode就会发生变化，导致明明set里有这个值，或者map里有这个key，但就是取不出来

kotlin中限制可变性三个手段
1. 使用val声明属性
2. 使用只读集合，List这些
3. 使用copy代替可变，data class自带

但要注意的是
1. val只针对绑定不可变，也就是说如果绑定背后是可变的话，val也拦不住，例如：
```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4)

var firstName = "liz"
var secondName = "chou"
val name get() = "$firstName $secondName"
firstName = "lizzz"
```
2. 只读集合也只是一个接口定义，仅仅是该接口不提供写方法而已，但是其背后的实例要是可读写也是拦不住的，例如：
```kotlin
val list: List<Int> = mutableListOf(1, 2, 3)
(list as MutableList<Int>).add(4)
```

在kotlin中，可变性主要来自两个地方
1. 使用var声明的变量
2. 可读写集合类型，MutableList这些（更准确来说是内部状态可变对象）

使用var声明的变量是绑定可变，就是这一刻是与某个值绑定，下一刻会与另一个值绑定。  
这种可变方式的好处是变化比较容易跟踪，通过设置getter/setter就行，而且可以通过明确的手段来限制并发修改冲突（例如同步锁）。

而可读写集合类型，主要是内部状态可变，但本身与变量之间的绑定关系是不变的。  
这种方式的好处是，性能好，不用重复创建对象，只是内部状态变更。但是监听变化，还有解决并发修改冲突就会麻烦一些，毕竟是人家内部的变化，硬要暴露出来，就需要重新实现这些集合类。

对于需要可变的代码，上面两种方式都可以。但是千万不能两种一起用，出问题都不知道是那个可变点导致的
```kotlin
var list = mutableListOf(1, 2, 3)
list.add(4)
// 或者
list = mutableListOf(1, 2, 3, 4)
```

最后，如果可变点是在自己控制的范围内，到问题不大。最糟糕的是这个可变点对外公开了，所有人都可以来修改你的状态，这样的代码就变得完全不可控的。
```kotlin
class Student(val name: String)
class School {
  private val students = mutableListOf<Student>()
  fun allStudents(): MutableList<Student> {
    return students
  }
}
```
这时候所有人都可能往你的学校里加学生，可变点泄露了。这里可以通过copy:
```kotlin
class Student(val name: String)
class School {
  private val students = mutableListOf<Student>()
  fun allStudents(): MutableList<Student> {
    return students.copy() // 防御性copy
  }
}
```
或者接口类型收窄来防御：
```kotlin
class Student(val name: String)
class School {
  private val students = mutableListOf<Student>()
  fun allStudents(): List<Student> {
    return students() // 收窄成只读List
  }
}
```

# 第三部份：效率问题

## 使所有操作背后的代价尽可能低

虽然现在获取机器性能的成本已经很低了，比程序员的薪酬成本低多了，但也难免会有规模效应。如果你的代码是在成千上万的移动设备上运行，性能差点，浪费的电量都可能足够一个小城市使用。更别说那些运行在成千上万服务器上的后台程序了，你的代码性能差点，就意味着需哟更多的机器，更高的运维成本，更容易出现的服务器瘫痪。

### 避免不必要的对象创建

创建对象本身就是一种成本，需要更多的内存，更多的cpu时间片，所以能省则省。

在JVM层面本身就会做很多优化来避免创建不必要的对象，例如相同的字符串对象和`Int`，`Long`等基础类型封装对象，在JVM底层会被复用：
```kotlin
val str1 = "liz"
val str2 = "liz"
println(str1 == str2) // true 值一样
println(str1 === str2) // true 引用的对象也一样，这就证明对象被复用了

val int1: Int? = 1 // 这里使用Int?是为了避免对象在JVM底层被优化成primitive了
val int2: Int? = 1
println(int1 == int2) // true 同样，值一样
println(int1 === int2) // true 引用的对象也一样，JVM在对待-128到127范围内的Int对象，都会自动复用

val int3: Int? = 128
val int4: Int? = 128
println(int3 == int4) // true 同样，值一样
println(int3 === int4) // false 但是因为超过了范围，引用的对象不一样了
```

创建对象之所以成本高，主要有
1. 对象存储的信息更多，占用空间更多，光是对象的头字段就至少需要16byte，加上对象自身的引用地址8byte，就需要至少24byte才能表达出一个对象。而相对应的，例如一个primitive int，只需要4byte，也就是说一个Int对象比primitive int多出了4倍的存储空间
2. 使用对象时，需要格外的寻址调用时间，需要通过应用地址，找到对象，并解出相应的值，每次访问对象都要做这些，也是一笔不少的成本
3. 创建对象时，需要在堆上申请内存空间，然后在栈上申请引用地址空间，相比于那些只在栈上申请存储空间的值，成本更高，更慢，更容易出错

如果你完全不使用对象，那么上面三种成本，你都不需要付出。不过一般情况下，你多不会这么做，不然你用C语言就好，像Java/Kotlin这样的语言，就是面向对象编程的。但是你还是可以选择对象复用，这样虽然在对象寻址调用时间上无法优化（毕竟你还是在使用对象）。但是因为创建的对象少了，占用的存储空间也就少了，申请存储空间的次数也少了，蚊子腿上的肉也是肉啊。

那有哪些方式可以实现对象复用呢

1. 使用`object`来声明单例对象。很多时候，对象本身只是表达一种状态，特别是对于不可变对象。对于这种状态对象，使用单例对象就可以很好的完成任务。例如我们要表达出一个链表是空的，由于空链表这种状态无论在那个链表上都是一样的，所以可以直接使用`object`方式来声明一个单例，当然这里还需要使用一些技巧来适配泛型的使用
```kotlin
sealed class LinkList<out T>

class Node<out T>(
  val head: T,
  val tail: LinkList<T>
) : LinkList<T>()

object Empty : LinkList<Nothing>()

/*
  由于这里使用的协变的泛型T，因此LinkList<Nothing>是所有其他ListList<*>的子类（Nothing是任意类型的子类），可以直接赋值任意Node<*>::tail

  而且由于Empty是单例，不会重复创建
 */

val list: LinkList<Int> =
  Node(1, Node(2, Node(3, Empty)))

val list2: ListList<String> =
  Node("A", Node("B", Node("C", Empty)))
```

不过要注意的是，对于可变对象，这个方式就玩不转了

2. 使用工厂方法，在方法内部实现对象复用。相比于直接调用构造函数来创建对象，使用工厂方法，对象创建的过程就更加的可控了。这里举两个例子：
    * 标准库中`emptyList`方法就是一个工厂方法，其内部会返回同一个`EMPTY_LIST`对象
    * 协程库中的`Dispatchers.Default`对象内部，维护了一个线程池，每次分配的时候都是通过一个工厂方法来完成，从线程池中取出一个空闲的线程对象，以达到复用的目的。类似的还有数据库连接池，原理都是大同小异，使用工厂方法来创建对象，以达到对象复用的目的

    当然，你的工厂方法可以更灵活一些，例如通过不同的参数可以创建不同的对象
    
```kotlin
private val connections = mutableMapOf<String, Connection>()

fun getConnection(host: String) =
  connections.getOrPut(host) { Connection(host) }

val con1 = getConnection("www.qq.com")
val con2 = getConnection("www.qq.com")
println(con1 == con2) // true 值相同
println(con1 === con2) // true 引用相对，对象被复用
```

这里的工厂方法，主要是使用了缓存的技巧，这种技巧除了可以用在创建任意不可变对象上，还可以用在纯函数的实现上。不过要注意的是，缓存这玩意，同时也意味着需要更新策略，淘汰掉过时的缓存，释放内存，不然很容易造成内存泄漏。当然这里可以引入像`SoftReference`和`WeakReference`这样的工具来解决gc的问题

3. 复杂对象声明提升。当你创建对象的过程复杂度很高，包括时间复杂度和空间复杂度，这时候，就不适合在一些多次调用的代码块里重复创建这个对象了，而是将对象创建的过程提升到代码块的外部，避免重复创建

```kotlin
fun <T: Comparable<T>> Iterable<T>.countMax(): Int
  = count { it == this.max() }

fun <T: Comparable<T>> Iterable<T>.countMax(): Int {
  val max = this.max() // 声明提升，count后的lambda会被循环调用，把max的计算提升到外部，能有效提升性能
  return count { it == max }
}
```

除了循环调用这种情况，有一种情况更容易被忽略，就是哪些工具函数，有的工具函数会被其他地方多次调用，当然也包括用在循环体里。如果这些工具函数里会重复创建复杂度高的对象（当然，这些对象本身是不可变的，和调用上下文无关的），这时候将对象的创建放在函数定义外部会更划算，同时也可以配合`lazy`函数实现惰性计算，下面的正则匹配就是一个很好的例子，因为众所周知，将字符串动态编译成一个正则表达式，是个复杂度很高的过程

```kotlin
private val IS_VALID_EMAIL_REGEX by lazy {
  "\\A(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\z".toRegex()
}

fun String.isValidIpAddress(): Boolean =
  matches(IS_VALID_EMAIL_REGEX)
```

4. 惰性计算，将对象创建的过程，延迟到第一次使用该对象的时候，上面提到的`lazy`函数，就是kotlin标准库提供给我们的用于实现惰性计算的工具。惰性计算使得原本是集中在一起的多个复杂度高的对象创建过程，分散开来，避免堆积。

5. 使用primitive代替对象。正如上文所说一般情况下JVM会自动帮我们做好这个优化。但是有的情况下，会迫使JVM将primitive退化成对象
    * 使用可空类型，例如`Int?`，因为primitive无法表达可空
    * 使用泛型，例如`Array<Int>`内部使用的是`Int`对象，而非primitive int，如果要使用primitive int，则需要改为使用`IntArray`
```kotlin
fun Iterable<Int>.maxOrNull(): Int? {
  var max: Int? = null // 由于这使用可空的Int?，导致JVM底层无法对其优化
  for (i in this) {
    max = if (i > (max ?: Int.MIN_VALUE)) i else max // 同时由于max可空，在每次使用时，都需要一次额外饿?:判断
  }
  return max
}

// 改为使用while，可以优化掉max的类型
fun Iterable<Int>.maxOrNull(): Int? {
  val iterator = iterator()
  if (!iterator.hasNext()) return null
  var max: Int = iterator.next() // 这里max直接就是Int类型，JVM底层会优化成primitive int
  while (iterator.hasNext()) {
    val e = iterator.next()
    if (max < e) max = e // 而且由于非空类型，这里max的使用更加简便
  }
  return max
}
```
    
不过要注意的是，这种性能上的差别只有在极端情况下，例如处理巨大的数值数组时，才会被体现出来，一般的业务逻辑处理，反而会因为过度优化导致可读写下降，上面的代码就是个很好的例子，如果你不认识迭代器，估计看都看不懂

### 在包含函数类型参数的函数上使用inline函数

这个标题有够复杂的，说白了就是
```kotlin
fun run(block: () -> Unit) {
  block()
}
// 最好写成
inline fun run(block: () -> Unit) {
  block()
}
```

这是由于JVM在底层实现函数对象的方式所导致的，首先你要搞清楚一个道理：
```kotlin
val fn: (Int) -> Unit
// 正式的表达
val fn: Function1<Int, Unit>
```
上面的两种表达是一个意思，`(Int) -> Unit`只是一种简写的语法糖，在JVM底层，他还是会被实现成类，而这个`Function1<Int, Unit>`类，是编译时编译器自动生成的，正常情况下你是看不见的，不过你可以试着这样写代码：

```kotlin
class OnClickHandler : () -> Unit {
  override fun invoke() {
    println("clicked")
  }
}
```

看到了吗，一个函数类型居然可以作为基类被继承，就足以证明他只是个语法糖，其本质就是个类。有了这个背景知识，你就不难理解，一个普通的lambda函数实际就是这个类的一个实例，而每次你将一个lambda函数作为参数传给另一个函数时，就是在创建一个对象，并将这个传递给这个函数作为参数。还记得我们上文说到的一个优化技巧就是“避免创建不必要的对象”吗，能连起来了吧。在普通函数上直接传入函数参数，就是在创建不必要的对象。

这种情况，当这个函数参数在创建时，通过闭包捕获了其他变量时，变得更加严重。因为这个时候不单止这个函数被转换成对象了，连那些被捕获的变量，都需要包裹在一个对象中，而且每次在函数中使用这个变量，都需要创建一个新的对象，这个开销是几何级增长的。

但是当我们使用inline来修饰这个外部函数时一切就不一样了，编译器会直接使用函数体替换掉函数的调用，而且这些传进去的函数参数，在其内部的调用，也会被他自身的函数体替换掉

```kotlin
inline fun repeat(time: Int, block: () -> Unit) {
  for (i in 0 until time) {
    block()
  }
}

repeat(10) {
  println("abc")
}
// 编译成
for (i in 0 until 10) {
  println("abc")
}
```

所有对象的创建都消失了，性能也就嘎嘎的了。

除了减少不必要的对象创建外，inline函数，还能配合`reified`关键字，实现类型信息保留。我们知道JVM在实现泛型时，为了向上兼容，在编译后的字节码中，进行了类型擦除，最直观的感受就是你不能在泛型函数中，使用`is T`表达式来判断类型
```kotlin
fun <T> isType(value: Any): Boolean {
    return value is T // Cannot check for instance of erased type: T
}
```
但是如果你使用inline函数，再配合`reified`关键字，神奇的一幕就发生了
```kotlin
inline fun <reified T> isType(value: Any): Boolean {
    return value is T // 编译通过了
}
println(isType<String>("liz")) // true
```
这是由于inline函数是使用函数体直接替换函数调用的，也就是运行时，还是在调用点的上下文中，这个地方的类型信息还是完整的，上面的代码会被编译成：
```kotlin
println("liz" is String) // 这当然能返回true呢
```

另外，在inline函数中，你还可以非常直观的使用`return`

这是由于在kotlin中，`return`返回的是最近的一个`fun`:
```kotlin
fun main() {
  repeat(10) {
    if (it > 5) return // 这里是直接return main的，也就是说下面的"done"不会被打印出来
    println(it)
  }
  println("done")
}
```

当我们不是以inline函数的方式来实现时，当中的lambda在编译后，就会被包裹在一个独立的对象中，这时候调用上下文就发生了变化，从原理上就无法`return`到`main`，这就不符合kotlin对`return`的定义了，因此，编译器会禁止这样的代码通过编译。

而使用inline函数，由于没有额外的对象创建，调用上下文也没有被切换，我们就能正常的`return`到`main`。因此编译通过了。

如果你只是想退出lambda，那在inline函数还是非inline函数中，你都可以通过`return@label`的方式返回

inline函数的缺点：
1. 不能使用递归，因为使用递归的话，编译器自己就会死循环，特别要注意的是哪些间接递归调用，会比较难发现
```kotlin
inline fun dec(num: Int) {
  if (num > 1) return dec(num - 1)
  return 0
}
dec(2)
// 编译成
if (2 > 1) return dec(1)
return 0
// 编译成
if (2 > 1) return {
  if (1 > 1) return dec(0)
  return 0
}
return 0
// 编译成
if (2 > 1) return {
  if (1 > 1) return {
    if (0 > 1) return dec(-1)
    return 0
  }
  return 0
}
return 0
// 无限循环下去
```
2. inline函数会直接暴露内部的实现，因此，对于一个`public`的inline函数，在内部就不能使用`private`或者`internal`级别的变量
3. inline函数使用不当，会使得生成的目标代码体积变大，特别是那种多层级联的inline函数。第一层调用10次，每次调用第二层10次，那么编译后，就会将第二层级的inline函数体展开成100次。体积增大了100倍。

最后，筒子们需要了解的就是在inline函数中，可以通过`crossinline`和`noninline`关键字，控制函数类型参数的行为。在正常的状态下inline函数的函数类参数也会被inline，而且可以在其内部使用`return`直接退出外层`fun`：
1. `crossinline`是用来限制，在这个函数类型参数内部，不能直接`return`，这在定义一些DSL时很有用
2. `noninline`是用来控制，这个函数类型参数不被inline，因为有的时候，这个函数类型参数要传递给别的函数，而这些函数有可能不接受inline，或者需要将这个函数类型参数存储在某个变量中，这时就不允许inline了

### 在需要将通用数据类型区别对待的地方，可以使用inline class

例如，同样是`Int`类型，在表达时间的时候，不同单位下，可能代表不同的意思，这时就需要区别对待，你可以选择将每种单位都封装成一个单独的类，例如
```kotlin
class Seconds(val seconds: Int)
class Millis(val milliseconds: Int)

var Int.seconds = Seconds(this)
var Int.milliseconds = Millis(this)

var time1 = 1.seconds
var time2 = 10.milliseconds
```
有了这样的封装就可以通过kotlin的类型系统来提供检查，避免用错时间单位。

但是前面我们已经说过，为了性能，我们应该避免创建不必要的对象，这里就产生了分歧。而inline class就是kotlin提供给我们的用于解决这种分歧的工具。

inline class从定义上就是将那种只在主构造函数中声明了一个属性的类，在使用时直接展开，而类上的方法，则编译成静态方法：
```kotlin
inline class Name(val name: String) {
  fun greeting() {
    println("hello $name")
  }
}
val name = Name("liz")
name.greeting()
// 编译后
val name = "liz"
Name.greeting(name)
```
有了这样的工具后，我们就可以在不增加运行时成本的情况下，使用kotlin的类型系统来增加代码的可维护性。

需要注意的是，这里的inline class语法其实已经被官方废弃，改为value class，具体可以参考官方文档，这里就不展开了，原理都是一样的。

### 删除掉过期的对象引用

这里就要先从GC大致的算法讲起，JVM通常采用的GC算法，都是基于一种叫可达性分析的算法，当一个对象不可达时，GC就会标记他，并以某种方式回收他所占用的内存。

而可达性分析，通常都是通过对象之间的引用关系来完成的，一般来说就是从根对象开始做递归遍历，能被查找到的都认为是可达的。

那么问题就来了，如果某个对象已经没有用了，但是我们意外的保存了对他的引用，这时候GC算法就会因为其“可达”而无法释法他的内存，最终导致内存泄漏。

因此这里所说的删除过期的对象引用，就是为了防止内存泄露的。但是说起来容易，做起来难，很多时候，我们会不自觉地保存了对象的引用，例如

1. 为了方便使用，将对象的引用保存在companion object，或者是其他全局object单例中，这里有一个需要特别注意的，就是在companion object或者全局object单例中保存了一个lambda，而lambda又通过闭包捕获到了某些对象，就这样这些对象的引用也就间接的被保存了起来，导致无法释放。所这里的解决方案也很直接，不要在这些地方保存对象引用，而是通过类似依赖管理的库，在需要时获取到所需对象的引用
2. 对象在逻辑上不可达，但是实际上其引用还是被保存了下来，导致内存没有被释放，例如
```kotlin
class Stack {
    private var elements: Array<Any?> =
        arrayOfNulls(DEFAULT_INITIAL_CAPACITY)
    private var size = 0

    fun push(e: Any) {
        ensureCapacity()
        elements[size++] = e
    }
    /*
      在pop操作时，通过size--，使得栈顶元素从逻辑上不可达，然而由于其应用还是被保存在了elements数组中，而从导致GC无法正常释放其内存
     */
    fun pop(): Any? {
        if (size == 0) {
            throw Exception("empty stack")
        }
        // 要解决这个问题很简单，就是将对象的引用修改成null即可
        return elements[--size]
    }
    
    private fun ensureCapacity() {
        if (elements.size == size) {
            elements = elements.copyOf(2 * size + 1)
        }
    }

    companion object {
        private const val DEFAULT_INITIAL_CAPACITY = 16
    }
}
```
3. 使用完的lambda函数，其内部通过闭包捕获的对象，导致对象的引用被保存了下来，导致内存没有被释放。这种情况通常出现在一个initial代码块中，例如lazy函数中的initial参数。和2一样，只需要将使用完的lambda修改成null即可解决，事实上，标准库中的lazy函数就是这样做的
4. 在一些带缓存逻辑的地方，cache里保存了过期对象的引用，可以使用Soft Reference/Weak Reference等工具来解决

## 高效地处理集合

### 在一些数据量巨大的集合中，需要多步处理时，使用Sequence来提高性能

Sequence是kotlin提供给我们的一种提高集合处理性能的工具，用起来很像RxJs/RxJava

1. 通常集合的处理都是一步一步的，例如对一个数组进行map然后filter，是对整个数组先进行map，得到一个新数组，再对新数组进行filter，又得到一个新数组。而Sequence则是按一个一个的，同样是对一个Sequence进行map然后filter，是将Sequence中的一个元素，先进行map，然后filter，接下来再对另一个元素进行map，然后filter。Sequence的这种处理方式，其实更接近我们常规使用单个循环顺序处理数组中所有元素的方式
2. 由于是按元素一个一个的去处理，一旦处理到某个元素时，结束条件满足了，Sequence就会停止处理其他元素，就类似我们在循环里，某个结束条件满足了就会break掉整个循环一样。这样会使得我们以更少的处理步骤来处理整个集合
3. 而且由于有这种提前结束的特性，即使Sequence对应的集合是个无限集合（例如通过生成器产生的集合）也不怕，只要结束条件满足，后续的元素都会被忽略，而不会产生无限循环的问题
4. 更重要的Sequence这种按一个一个的处理方式，不会产生中间状态的新集合，内存使用更加合理，这就是标题所说的，在处理一些数据量巨大的集合时，使用Sequence能提高性能的重要原因。例如我们在处理一个大文件时，就可以使用Sequence。kotlin在处理文件的方法上，也提供了对Sequence的支持，例如`useLines`方法

那么有没有什么地方是Sequence不适合的呢？有的，如果处理中包含了sorted操作，由于现在对Sequence使用sorted，其内部会先将Sequence重新整合成List，然后再调用java标准库提供的sorted方法，这样就导致了Sequence本来的优势消失了，另外由于要整合成List，原本的Sequence也不能是无限的了。

最后如果我们要对Sequence进行调试，可以使用ide提供的"Kotlin Sequence Debugger"工具

### 使用kotlin标准库提供的函数，来聚合对集合的操作

1. filterNotNull
```kotlin
val list: List<Int?> = listOf(1, 2, null, 3)
val list2 = list
  .filter { it != null}
  .map { it!! }
// 下面的操作效果是一样的，但少了一个中间步骤
val list3 = list.filterNotNull()
```
2. mapNotNull
```kotlin
val list = listOf(1, 2, 3)
val list2 = list
  .map { if (it % 2 == 0) it else null }
  .filterNotNull()
// 下面的操作效果是一样的，但少了一个中间步骤
val list3 = list.mapNotNull { if (it % 2 == 0) it else null }
```
3. joinToString
```kotlin
val list = listOf("1", "2")
val str1 = list
  .map { "f$it" }
  .joinToString()
// 下面的操作效果是一样的，但少了一个中间步骤
val str2 = list.joinToString { "f$it" }
```
4. filterIsInstance
5. sortedWith
6. listOfNotNull
7. filterIndexed, mapIndexed, reduceIndexed, foldIndexed, forEachIndexed

### 在需要处理包含大量基础类型数据的集合时，尝试改为primitive array

这里所说的primitive array，就是指IntArray，LongArray这些，但是需要注意的是，这些优化都是很底层的，对于一般情况而言，不会有什么效果，甚至会降低代码的可读性，因此只能在那些真正的性能瓶颈处使用，例如视频游戏或图像处理这样的地方

### 使用可读写集合来提高性能

虽然我们在最开始的时候就说过，要限制可变性。但是可变性的确在性能上更好一些，特别是在处理一些数据量大的集合时，如果是用只读集合，那么每次更新时都需要先将原来的集合先复制一遍，而使用可读写集合则没有这个问题。

所以我们从来都没有说不使用可变性，而是要对其使用的地方有所限制，做到能不用就不用。但是在一些函数的内部实现上，由于不会对外泄露可变性，加上性能上的考虑，还是可以引入可变性的，kotlin的标准库，其实也是这样做的，最后在返回结果时，再通过接口收窄的方式，限制对外的可变性