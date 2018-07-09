最近在研究正则表达式的底层实现原理，在这里先总结一下：


## 正则表达式的特点

* 正则表达式中的字符分为 **字符字面量** 和 **元字符**
    > * 不同的正则表达式引擎对不同的元字符支持程度不同，能提供的匹配能力也不同
    > * 不同的正则表达式引擎对不同的字符编码支持程度不同，对相同字符的处理也不同

* 正则表达式引擎从实现原理来分，可以分为 **基于表达式(nfa)** 和 **基于文本(dfa)** 两种
    > * 从匹配能力来说，基于nfa的正则表达式引擎提供的元字符更多(分配捕获，反向引用，惰性量词等)，功能更强大一些
    > * 从匹配效率来说，基于dfa的正则表达式引擎因为不需要回溯，效率要更高一些
    > * 现代的正则表达式引擎内部会同时包含基于nfa和dfa两个引擎，并根据不同情况选择使用不同的引擎

## 正则表达式的基本准则

* 优先匹配最左
    > 对于字符串中多处可能匹配的文本，正则表达式总是优先匹配最左的文本
* 标准量词总是贪婪
    > 对于标准量词(`?`, `*`, `+`)，其表现总是贪婪的，也就是匹配优先的，只有当后续的表达式匹配失败，才会“归还”已匹配字符

## 正则表达式支持的元字符
> 注意，这里提及的元字符并没有区分是nfa引擎还是dfa引擎的，其中打勾的是javascript中支持的元字符

### 特殊字符
* `\n` 换行符 ✔️
* `\r` 回车符 ✔️
* `\t` 水平制表符 ✔️
* `\v` 垂直制表符 ✔️
* `\a` 报警符
* `\b` 退格符 ✔️
* `\e` Esc符
* `\f` 分页符 ✔️
* `\0` Null字符 ✔️

### 八进制字符
* `\num` 八进制字符

### 十六进制字符
* `\xnum` 十六进制字符 ✔️
* `\x{num}` 十六进制字符

### Unicode字符
* `\unum` Unicode字符 ✔️
* `\Unum` Unicode字符

### 普通字符组
* `[a-z]` 肯定字符组 ✔️
* `[^a-z]` 否定字符组 ✔️

### 特殊字符组
* `.` 匹配除换行符外任意字符 ✔️
* `\s` 匹配空白字符 ✔️
* `\d` 匹配数字字符，等价于[0-9] ✔️
* `\w` 匹配字母字符，等价于[^0-9] ✔️
* `\S` 匹配非空白字符 ✔️
* `\D` 匹配非数字字符，等价于[A-Za-z0-9_] ✔️
* `\W` 匹配非字母字符，等价于[^A-Za-z0-9_] ✔️

### 字节字符
* `\C` 单字节字符
* `\X` Unicode组合字符序列

### 零宽断言
* `^` 字符串起始/换行符之后 ✔️
* `\A` 字符串起始
* `$` 字符串结束/换行符之前 ✔️
* `\Z` `\z` 字符串结束
* `\G` 上次匹配点
* `\b` 单词分割点 ✔️
* `\B` 非单词分割点 ✔️
* `\<` `\>` 单词分割点
* `{?=...}` 顺序肯定环视 ✔️
* `{?!...}` 顺序否定环视 ✔️
* `{?<=...}` 逆序肯定环视
* `{?<!...}` 逆序否定环视

### 分组
* `(...)` 分组捕获 ✔️
* `\1` `\2` `\3` 反向引用
* `(?:...)` 分组不捕获 ✔️
* `(?<name>...)` 命名分组
* `(?>...)` 固化分组

### 匹配优先(贪婪)量词
* `?` 匹配0次或1次 ✔️
* `*` 匹配0次或以上 ✔️
* `+` 匹配1次还以上 ✔️
* `{min, max}` 匹配最少min次，最多max次 ✔️

### 忽略优先(惰性)量词
* `??` 匹配0次或1次 ✔️
* `*?` 匹配0次或以上 ✔️
* `+?` 匹配1次还以上 ✔️
* `{min, max}?` 匹配最少min次，最多max次

### 占有优先量词
* `?+` 匹配0次或1次
* `*+` 匹配0次或以上
* `++` 匹配1次还以上
* `{min, max}+` 匹配最少min次，最多max次

### 多选结构
* `...|...|...` 多个匹配分支 ✔️

上面总结了很多，但对于要完成理解正则，光有这些还不足够。我们需要自己动手实现一个“正则表达式引擎”。这里之所以要用引号，主要的原因是要完成实现一个功能完备的正则表达式引擎，需要上千行代码，而其中真正对于我们理解正则表达式匹配原理有帮助的代码只占其中很少的一部分。

因此，我们这里要实现的正则表达式引擎，只是一个完整的引擎的一个很小的子集。其中能匹配的元字符包括`^`，`$`，`.`，`?`，`*`，`+`，`\w`，`\d`，`[...]`。甚至我们都没有用上nfa和dfa这样的数学模型，而仅仅使用了最简单的dfs搜索算法。你可以把字符串匹配想象成一个文本搜索过程，算法会遍历整个正则表达式，并在每一步上尝试对当前对应字符进行匹配，如果匹配成功，则继续进行下一步的匹配，如果匹配不成功，则会回退到上一步，并继续尝试其他可行的分支（该过程也就是nfa中说的回溯）。

这里的实现是参考Andy Oram和Greg Wilson编写的《代码之美》中实现正则表达式引擎，具体代码如下：

```typescript
interface IValue {
    equal(c: string): boolean
}

interface IToken {
    isMetaCharacter(c: string): boolean
    isCharacter(c: string): boolean
    isMatch(c: string): boolean
}

class Value implements IValue {
    private value_: string

    constructor(value: string) {
        this.value_ = value
    }

    equal(c: string): boolean {
        return this.value_.indexOf(c) > -1
    }
}

class Token implements IToken {
    private meta_: boolean
    private value_: IValue

    constructor(value: string, meta: boolean) {
        this.meta_ = meta
        this.value_ = new Value(value)
    }

    isMetaCharacter(c: string): boolean {
        return this.meta_ && this.value_.equal(c)
    }

    isCharacter(c: string): boolean {
        return !this.meta_ && this.value_.equal(c)
    }

    isMatch(c: string): boolean {
        if (this.isMetaCharacter('.')) {
            return c !== undefined
        } else if (this.isMetaCharacter('w')) {
            return '_' == c ||
                   'a' <= c && 'z' >= c ||
                   'A' <= c && 'Z' >= c ||
                   '0' <= c && '9' >= c
        } else if (this.isMetaCharacter('d')) {
            return '0' <= c && '9' >= c
        } else {
            return this.isCharacter(c)
        }
    }
}

class Matcher {
    private tokens_: IToken[]
    private target_: string

    constructor(tokens: IToken[], target: string) {
        this.tokens_ = tokens
        this.target_ = target
    }

    private matchQuestion_(i: number, j: number): boolean {
        const token = this.tokens_[i]
        const input = this.target_[j]

        if (token.isMatch(input) && this.match_(i + 2, j + 1)) {
            return true
        } else {
            // 否则跳出x?的匹配过程
            return this.match_(i + 2, j)
        }
    }

    private matchStar_(i: number, j: number): boolean {
        const token = this.tokens_[i]
        const input = this.target_[j]

        if (token.isMatch(input) && this.matchStar_(i, j + 1)) {
            return true
        } else {
            // 否则跳出x*的匹配过程
            return this.match_(i + 2, j)
        }
    }

    private matchPlus_(i: number, j: number): boolean {
        const token = this.tokens_[i]
        const input = this.target_[j]

        if (token.isMatch(input) && this.matchStar_(i, j + 1)) {
            return true
        } else {
            // x+必须匹配至少一次
            return false
        }
    }

    private match_(i: number, j: number): boolean {
        const token = this.tokens_[i]
        const tokenNext = this.tokens_[i + 1]
        const input = this.target_[j]

        if (i >= this.tokens_.length) {
            // 如果已经到正则的末尾，证明已经匹配成功
            return true
        } else if (token.isMetaCharacter('$') && !tokenNext) {
            // 如果正则中包含$，并且匹配到字符串末尾，则匹配成功
            return j === this.target_.length
        } else if (tokenNext && tokenNext.isMetaCharacter('?')) {
            // 如果正则中包含x?，则需要进行特殊处理
            return this.matchQuestion_(i, j)
        } else if (tokenNext && tokenNext.isMetaCharacter('*')) {
            // 如果正则中包含x*，则需要进行特殊处理
            return this.matchStar_(i, j)
        } else if (tokenNext && tokenNext.isMetaCharacter('+')) {
            // 如果正则中包含x+，则需要进行特殊处理
            return this.matchPlus_(i, j)
        } else if (token.isMatch(input)) {
            // 如果正则中的字符与字符串中对应位置的字符相等，则匹配下一位
            return this.match_(i + 1, j + 1)
        }

        return false
    }

    // 支持的元字符包括^$.*+?
    match(): boolean {
        const tokens = this.tokens_
        const target = this.target_

        if (tokens.length === 0) {
            // 如果正则的长度为0，则应该能匹配所有的字符串
            return true
        } else if (tokens[0].isMetaCharacter('^')) {
            // 如果正则以^开头，则只从字符串的起始位置开始匹配
            return this.match_(1, 0)
        } else {
            // 从字符串的起始位置开始尝试匹配，如果不行，就后移一位
            for (let i = 0; i < target.length; i++) {
                if (this.match_(0, i)) {
                    return true
                }
            }
        }

        return false
    }
}

export class Pattern {
    private regexp_: string
    private tokens_: IToken[] | null = null

    constructor(regexp: string) {
        this.regexp_ = regexp
    }

    private compile_(): void {
        const regexp = this.regexp_
        const tokens = []

        let i = 0

        while (i < regexp.length) {
            let value = regexp[i]
            let meta = false

            if ('\\' === value) {
                const pos = i + 1
                
                value = regexp[pos]
                meta = 'wd'.indexOf(value) > -1
                i = pos
            } else if ('[' === value) {
                const start = i + 1
                const end = regexp.indexOf(']', start)

                value = regexp.slice(start, end)
                i = end
            } else if ('^' === value) {
                meta = i === 0
            } else if ('$' === value) {
                meta = i === regexp.length - 1
            } else if ('?*+.'.indexOf(value) > -1) {
                meta = true
            }

            tokens.push(new Token(value, meta))

            i++
        }

        this.tokens_ = tokens
    }

    match(target: string): boolean {
        // 惰性计算
        if (this.tokens_ === null) {
            this.compile_()
        }

        return (new Matcher(this.tokens_ as IToken[], target)).match()
    }
}
```

代码中主要分成了两个主要的类`Pattern`和`Matcher`。`Pattern`类可以理解为js当中的`RegExp`类，主要负责把代表正则表达式的字符串编译成易于进行匹配的形式。在这里我们把他简单的编译成`Token`序列，如果你的正则表达式引擎是基于nfa/dfa的，就需要把正则表达式编译成对应的状态自动机。

在调用`Pattern`类的`match`方法事，程序首先会创建一个`Matcher`类，用于保存匹配过程中所有需要跟踪的状态，然后调用其`match`方法开始匹配。在匹配过程中，我们会利用dfs算法，遍历整个`Token`序列，搜索字符串中与表达式匹配的部分。而回溯主要发生在`matchQuestion`，`matchStar`和`matchPlus`三个方法中。一旦匹配成功，`match`方法就会结束匹配过程并马上返回。

根据这份代码，我们就很容易理解为什么基于nfa的正则表达式引擎在某些情况下性能会下降。我们可以考虑以下这个正则表达式：

```javascript
'https?://.*\\.qq\\.com/.*'
```

这个正则表达式主要用于判断url是否某个域名之下的合法url，这在web应用开发中是非常常见的。利用我们上面实现正则表达式引擎，我们尝试匹配一下这个url：

```javascript
`https://mp.weixin.qq.com/s?__biz=MzAxNzUzNDIwMg==&mid=2653529185&idx=1&sn=fddafcdd02c0613ab81b773dbb85933d&chksm=803919f4b74e90e2751c0aea795a9ad19093085fe9b487a35885d628629846465179d94c19b1&mpshare=1&scene=23&srcid=0416ncnUJSMyYBiE06ejF2Ch#rd`
```

我们很快就会发现当匹配到`.*`这个部分时，引擎会直接匹配到字符串的末端，并报告匹配失败，同时进行回溯，把字符串中最后的字符`'d'`释放，然后重新进行匹配。这时匹配仍然会失败，引擎再次回溯，并释放字符串中的倒数第二个字符`'r'`，这样的尝试匹配-匹配失败-回溯的过程会一直持续，直到引擎回到这个位置：

```javascript
'https://mp.weixin.qq.com/...'
                  ^
```

这时，引擎再次尝试匹配，这次终于匹配成功，并最终返回`true`。

那如果我们要解决以上问题要怎么做呢？方法有很多，其中我们可以把`.*`部分替换成惰性匹配`.*?`，这时引擎就会优先跳过该部分的匹配，并尝试匹配表达式中后续的部分，只有在匹配不成功的情况下，尝试匹配`.*?`。

以上～