# 如何利用web技术实现一个terminal

如果有使用过vscode或者hyper这样的工具的同学，应该会发现其内部都实现了一个ternimal，而且这些应用都有一个共同点，就是他们都是基于electron的，也就是说它们实际都是我们平时看到的web页面。

作为一个前端开发，一直都很好奇他们到底是怎么实现的，趁这个周末在家闲着就翻看了一下他们的代码，这时才知道要实现这样一个功能，实际并没有我们想象中的难，应该说最难的部分，已经有人帮我们实现了，我们只需要使用正确的工具，就可以快速的开发一个自己的基于web的ternimal。

为了尽量保持简单，我这里假设代码是跑在一个electron环境的renderer进程中，如果你对electron有任何疑问，可以直接查看[官网](https://electronjs.org/)

要实现一个web版的ternimal实际上我们需要解决两个问题：

* 第一，是js如何与系统通信，向系统发送命令，并获得其响应。例如我们输入`ls -l`，系统把当前路径下的文件列表信息返回给我们
* 第二，是获得了系统的响应，我们要如何显示到页面上，其中我们要处理中英文混排，颜色，光标等信息

### 在web上显示终端信息
我们首先来处理第二个问题。假设我们已经获取到系统的响应，要如何显示在页面上。最简单的方法当然就是直接显示咯，有什么就直接输出什么到页面上。但这种方法实际是有很多问题的。第一，一般来说终端中要显示的文本行数是非常多的，例如你要查询一个日志文件，成千上万行文本都是小事，直接显示肯定是不行的。

那我们就换个方法，维护着一个数组，每一项保存一行文本，并记录当前屏幕显示的开始行下标和屏幕中显示的总行数，每次输入都会刷行这个数组，并且只显示屏幕中实际需要显示的部分：

```javascript
const rows = []
const startIdx = 0
const currentIdx = 0
const diplayRows = 80

function write(char) {
    if (char.match(/[\r\n]/)) {
        rows[++currentIdx] = ''
    } eles {
        rows[currentIdx] += char
    }
    render()
}

function render() {
    let output = ''

    for (let idx = 0; idx < diplayRows; idx++) {
        output += rows[startIdx + idx] + '\n'
    }

    // 这里display假设为我们用于显示的textarea节点
    display.value = output.slice(0, -1)
}
```

但这样做也有其他问题，例如一般终端的显示实际是包含一些控制字符的（例如以`\x1b`开头的字符，用于控制光标的位置，控制字体的颜色，背景的颜色），这些控制字符直接显示也是不合理的。

当然除了上面说到的一些问题，还有诸如中英文混排，光标显示等一系列问题需要处理。所幸的是，这些工作`xterm`都已经帮我们实现好了，我们只要安装`xterm`，并把它挂载到我们的dom节点上即可，首先我们需要安装`xterm`：

```shell
npm i xterm
```

然后我们在代码中初始化`xterm`，并且挂载到我们指定的dom节点上：

```javascript
const { Terminal } = require('xterm')
const fit = require('xterm/lib/addons/fit/fit')

const xterm = new Terminal({ cursorBlink: true })

xterm.open(document.querySelector('#terminal'))
xterm.fit()
```

### js与系统通信

到这里为止，我们的第二个问题已经解决了。接下来我们将着手解决第一个问题。要使得js和系统通信，能想到最简单的方式是，通过`child_process`模块的`spawn`方法，把我们的输入作为第一个参数输入，并捕获其输出：

```javascript
const spawn = require('child_process').spawn

function read(command) {
    return new Promise((resolve, reject) => {
        const cp = spawn(command)

        let output = ''
        let error = ''

        cp.stdout.on('data', (data) => {
            output += data.toString('utf-8')
        })

        cp.stdout.on('end', () => {
            resolve(output)
        })

        cp.stderr.on('data', (data) => {
            error += data
        })

        cp.stderr.on('end', () => {
            reject(error)
        })
    })
}

(async () => {
    try {
        xterm.write(await read(command))
    } catch (error) {
        xterm.write(error)
    }
})()
```

但这种方法在每次输入一条命令的时候，都需要创建一个新的子进程，更好的方法是直接创建一个可以交互的进程（例如在mac下我们可以直接创建一个`bash`进程），每次都把用户的输入写入到该进程的`stdin`中：

```javascript
const spawn = require('child_process').spawn
const bash = spawn('bash')

bash.stdout.on('data', (data) => {
    xterm.write(data)
})

bash.stderr.on('data', (data) => {
    xterm.write(data)
})

bash.stdin.write(command)
```

这样的确可以实现交互式的输入输出，但这里有两个问题
* 第一，输入的内容没有回显，需要我们编写代码把用户的输入显示到xterm中
* 第二，输出的内容，由于没有可显示区域的概念，因此会出现排版错乱的问题

为了解决这些问题，我们不能简单创建一个`bash`进程，而是要创建一个`bash`的虚拟终端（pty）进程，而为了创建pty进程，我们需要调用系统提供的`forkpty`方法，具体可以参看[这里](https://www.gnu.org/software/gnulib/manual/html_node/forkpty.html)。而node中并没有提供这样的封装，因此我们需要另外安装`node-pty`模块，该模块是对`forkpty(3)`接口的封装。

不过安装的过程并不是十分顺利，由于`node-pty`是需要使用`node-gyp`进行本地编译的，但由于electron中node的版本与我本地的node版本并不一样，因此需要额外对`node-pty`重新编译，这里使用的是electron-rebuild工具，它会自动根据electron的版本，对node模块进行重新编译（过程中需要科学上网）：

```shell
electron-rebuild -f -w /node_modules/node-pty
```

在安装完，并重新编译以后，我们就可以利用它来创建一个`bash`的虚拟终端进程：

```javascript
const spawn = require('node-pty').spawn
const bash = spawn('bash', [], {
    name: 'xterm-color',
    cols: xterm.cols,
    rows: xterm.rows,
    cwd: process.cwd(),
    env: process.env
})

bash.on('data', (data) => {
    xterm.write(data)
})

xterm.on('data', (data) => {
    bash.write(data)
})
```

到这里为止，我们已经可以在web页面中与终端进行交互了。当然上文说的都是在electron的renderer进程中运行的代码，如果你需要在浏览器中运行，则另外需要诸如web socket的辅助。

以上～～