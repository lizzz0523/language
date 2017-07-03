最近在开发一个cli小工具时，遇到了一些关于创建子进程的问题，因此在这里记录一下nodejs和`child_process`模块相关的一些知识点，主要参考的是nodejs的[官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/child_process.html)，可以理解为是官方文档的意译。

nodejs里的`child_process`模块主要提供了与`popen`系统调用类似但不完全一样的，创建子进程的能力，通过调用`child_process`里的`spawn`方法，我们就可以在nodejs里创建子进程，并且通过相关的stdio来进行进程之间的通信：

```javascript
const spawn = require("child_process").spawn;
const child = spawn("ls", ["-lh", "/usr"]);

child.stdout.on("data", (data) => {
  console.log(`stdout: ${ data }`);
});

child.stderr.on("data", (data) => {
  console.log(`stderr: ${ data }`);
});

child.on("close", (code) => {
  console.log(`child process exited with code ${ code }`);
});
```

上面的例子中，我们创建了运行`ls`命令的子进程，并且传入参数`-ls /usr`，要注意的是，这里nodejs是创建子进程，直接运行`ls`命令，而非先创建`shell`进程，然后在`shell`中运行`ls`命令，前者效率更高。该行为可以通过在`spawn`方法的第三个参数中传入`{shell: true}`来重置。

返回的`child`是一个`ChildProcess`实例，通过监听`child`上相关stdio的`data`事件，可以获取到子进程的io流，所有的stdio都是`Stream`对象。通常情况下通过`stdin`输入流向子进程写入数据时，子进程会马上相应，除非子进程在内部使用了line-buffered。

一旦子进程退出，例如
* 子进程报错
* 子进程调用process.exit()
* 父进程调用child.kill()

就会抛出`close`事件。

除了`spawn`方法，nodejs还在其上针对不同的应用场景，封装了不同接口：
* `spawn(file, args, options)` 创建子进程，直接运行file
* `execFile(file, args, options, callback)` 创建子进程，直接运行file
* `exec(command, options, callback)` 创建shell子进程，然后在shell中执行command
* `fork(script, args, options)` 创建node子进程，载入module模块

这些方法（包括`spawn`）均为异步执行，同时nodejs还提供了其同步的版本，方便在一些像cli工具里执行：
* `spawnSync`
* `execFileSync`
* `execSync`

接下来，我们逐个过一下`child_process`给我们提供的这些方法，这里主要记录的是异步方法，对于相应的同步方法，其参数和功能与其相应的异步版本是一样的。

## spawn(file, args, options)
`spawn`方法是`child_process`的核心方法，所有其他的方法，都是`spawn`之上的封装，因此了解了`spawn`方法，基本就等于掌握的`child_process`模块。

`spawn`方法，主要功能是启动一个新的子进程，并且在该子进程中，运行一个可执行文件`file`，并且传入参数`args`。这里我们以执行`ls`命令为例，我们知道`ls`命令实际是在`/bin`目录下的`ls`文件，功能是查看某个路径下的文件信息。在执行`ls`命令时，可以通过传入一个参数，来告诉`ls`命令要查看的路径：
```javascript
const spawn = require("child_process").spawn;

spawn("ls", ["/usr"], { stdio: "inherit" });
```
其运行的结果就是：
```bash
$ node spawn.js 
bin  etc  games  include  lib  lib64  libexec  local  nmagent  sbin  share  src  tmp
```

如果我们想实现`ls /usr | grep lib`这样的复合命令，可以通过创建两个子进程，通过stdio的管道功能来实现：
```javascript
const spawn = require("child_process").spawn;

const ps = spawn("ls", ["/usr"]);
const grep = spawn("grep", ["lib"], { stdio: ["pipe", "inherit", "inherit"] });

ps.stdout.pipe(grep.stdin);
```
其运行的结果是：
```bash
$ node spawn.js
lib
lib64
libexec
```

接下来我们看看参数:

### file: String
`file`指定了子进程需要运行的可执行文件，在上面的例子，就是`/bin/ls`。但由于`/bin`目录子在系统路径中，因此，直接传入`ls`是一样的。

### args: Array
`args`是传递给可执行文件的参数列表。如果是nodejs的可执行文件，可以在`process.argv`中读取相关参数。

### options: Object
`options`参数是配置项，通过不同的配置，可以修改`spawn`的默认行为。

#### options.cwd: String
`cwd`是用于配置子进程执行时所在的工作目录，默认在当前工作目录。这里举个例子，假设我们有如下目录结构：
```bash
.
|-- sub
|   `-- hello
`-- spawn.js
```
其中，`hello`是一个可执行文件，其作用是输出一段`hello world`文本。而我们的spawn.js文件内容如下：
```javascript
const spawn = require("child_process").spawn;
const child = spawn("./hello", [], { cwd: `${ __dirname }/sub`, stdio: "inherit" });

child.on("error", (err) => {
    console.log(err);
});
```
可以看到，通过`cwd`字段，我们把当前工作目录转换到了`sub`文件夹下。执行spawn.js文件，得出如下结果：
```bash
$ node spawn.js 
hello world
```
如果这里没有设置`cwd`，那么我们将会接受到`error`事件。

#### options.env: Object
`env`是用于向子进程传递环境变量的，默认值是`process.env`。这个没有什么好解释的吧？

#### options.shell: Boolean | String
之前在讲解`spawn`方法时，特意说明了，`spawn`是直接执行`file`文件的，而不会额外去创建一个shell进程，再执行`file`。但我们可以通过指定这个`shell`选项为`true`（默认为`false`），来重置该行为。此时`file`参数就会变成shell要执行的命令，任何shell能解析的字符串都可以使用，包括文件通配符等，因此出于安全的考虑，任何用户的输入都不要轻易作为`file`参数传入。

除了可以传入`true`以外，`shell`选项还可以传入一个字符串，用来指定shell解析器。

#### options.argv0: String
文档里说是设置子进程的`argv[0]`变量的值，默认情况下，改值为`file`的值。其实应该是挺好理解的，然而并没有什么卵用。我分别试了shell，nodejs，python三种常用的脚本，这个`argv0`选项完全没有起到作用，唯独采用c编写的程序能正确设置。而在nodejs中，可以通过`process.argv0`来获得，看来脚本语言都已经在内部处理掉这个argv[0]参数了。

#### options.stdio: String | Array
`stdio`是用于配置子进程的io行为的选项。

这里首先说明一下，这个`stdio`实际是一个数组，数组的每一位分别是用于设置[`stdin`, `stdout`, `stderr`]。如果传入的是一个字符串而非数组，则表示数组中的三个值设置为该字符串。

那么`stdio`有那些值可选呢：
* `pipe` 

该值为`stdio`的默认值，当`stdio`设置为`pipe`字符串时，nodejs就会在父进程和子进程之间，创建一个管道。而管道在父进程一端，会通过`child.stdin`，`child.stdout`，`child.stderr`暴露给父进程调用。这也是我们之前的例子中使用到的方法。

* `ignore`

当`stdio`设置为`ignore`字符串时，nodejs会把子进程的io重定向到`/dev/null`中。也就是忽略其io。

* `Stream`对象

如果我们希望子进程的io被重定向到指定的文件中，例如作为日志的输出，我们可以通过传入一个`Stream`对象给`stdio`来完成。例如：
```javascript
const fs = require("fs");
const spawn = require("child_process").spawn;

const stdout = fs.createWriteStream(`${ __dirname }/log.txt`);

stdout.on("open", () => {
    spawn("ls", ["/usr"], { stdio: [ null, stdout, stdout ] });
});
```
这时子进程的所有输出，都会流入到log.txt文件中：
```bash
$ cat log.txt 
bin
etc
games
include
lib
lib64
libexec
local
nmagent
sbin
share
src
tmp
```
或者如果你希望复用父进程的io，则可以通过传入：
```javascript
spawn("ls", ["/usr"], { stdio: [ process.stdin, process.stdout, process.stderr ] });
```
或者把`stdio`设置为`inherit`。

*  正整数 

和`Stream`对象相似，可以传入一个在父进程中已经打开的文件描述符，例如`process.stdin`的文件描述符为0，`process.stdout`的文件描述符为1，`process.stderr`的文件描述符为2。也就是说如果你传入：
```javascript
spawn("ls", ["/usr"], { stdio: [ 0, 1, 2 ] });
```
和把`stdio`设置为`inherit`是一样的效果。

* `ipc` 

最后如果把`stdio`设置为`ipc`字符串，那么nodejs就会在父子进程中，建立起一个ipc通道。通过这个ipc通道，父子进程可以进行双工通信，通信的内容可以是JSON数据或者是文件描述符（也就是可以通过ipc传递socket）。

在父进程，我们可以通过调用`child.send`方法来向子进程发送消息。而在子进程（假设该子进程是一个nodejs程序），则可以通过调用`process.send`方法来向父进程发送消息。而在父子进程，都可以通过监听`message`事件来捕获对应的消息：
```javascript
// parent.js
const spawn = require("child_process").spawn;
const child = spawn("node", ["./child.js"], { stdio: [0, 1, 2, "ipc"] });

child.send({ "hello": "world" });

child.on("message", (message) => {
    console.log(message);
});

// child.js
process.on("message", (message) => {
    process.send({ "hi": message.hello });
});
```
执行结果为：
```bash
$ node parent.js 
{ hi: 'world' }
```

#### options.detached: Boolean
`detached`选项在不同平台上有不同的意思。在win32平台上，当`detached`设置为`true`时，子进程在父进程退出后，仍然能继续运行。子进程拥有自己的输出控制台。而且该选项一旦设置了，不能取消。

在unix平台上，则比较复杂，这里涉及到进程组和进程会话，为了不扯太远，这里就不作过多的解析，有兴趣的同学可以看看[这篇文章](http://blog.lucode.net/linux/posix-process-group-note.html)。我们只要知道，在unix平台，当`detached`设置为`true`时，新创建的子进程，会被放进新的进程会话中的新的进程组内，并且作为该进程组的组长（即进程组id等于该子进程id）即可，也就是说新的子进程和父进程之间没有任何关联。这里写个例子来说明一下：
```javascript
// parent.js
const spawn = require("child_process").spawn;
const child = spawn("node", ["child.js"], {
    detached: false,
});

// child.js
// 这里是主要用于维持子进程不退出，方便调用ps命令才看进程状态
setInterval(() => {
    console.log("test");
});
```
在终端调用`ps -eo pid,ppid,gpid,cmd | grep node`：
```bash
$ ps -eo pid,ppid,pgid,cmd | grep node
 3985 19305  3985 node parent.js
 3991  3985  3985 node child.js
```
可以看出来，此时两个进程的进程组id（pgid）是一样的。
如果我们设置`detached`为`true`，则输出结果为：
```bash
$ ps -eo pid,ppid,pgid,cmd | grep node
 3511 19305  3511 node parent.js
 3517  3511  3517 node child.js
```
此时，父进程和子进程的进程组id不同了，而且子进程的进程组id就等于自己的进程id（pid），也就是新的子进程独立成组，并且成为该进程组的组长。

但是在unix平台，光靠设置`detached`为`true`并不能使子进程成为守护进程，就像win32平台那样。在unix平台上，我们还需要额外完成两个步骤
* 设置`stdio`为`ignore`或者重定向到其他文件中，也就是使子进程的io和父进程的io独立开来
* 调用`child.unref`方法来告诉nodejs，不需要在父进程的事件循环中引用子进程，父进程可以自由退出。

下面是一个启动守护进程的例子：
```javascript
const spawn = require("child_process").spawn;

const child = spawn("node", ["child.js"], {
    detached: true,
    stdio: "ignore"
});

child.unref();
```

以上这些就是`spawn`方法的全部介绍了。基本掌握了这些内容，就等于掌握了`child_process`模块。

对于`child_process`其他方法，这里就做一些简单介绍

### execFile(file, args, options, callback)
这个方法和`spawn`方法的用法是基本一致的，只有两个地方不同
* `execFile`方法中没有`shell`选项，或者可以理解为`shell`选项写死成false了，也就是说，`execFile`方法是直接运行一个可执行文件，而不可以运行shell命令。
* `execFile`方法有`callback`回调函数，所有的子进程输出（stdout，stderr）都会被收集和解码，作为`callback`的参数回传。而解码的字符集可以通过`encoding`选项设置，解码的buffer大小可以通过`maxBuffer`来控制。

### exec(command, options, callback)
这个方法和`execFile`基本是一样的，同样是提供了`callback`回调函数，同样是没有`shell`选项，而不同的地方就是这个`shell`选项被写死成`true`了，也就是说，`exec`方法是先启动一个shell进程，然后在这个shell进程内执行`command`命令。

### fork(script, args, options)
`fork`方法则和前两个方法有一点不同，`fork`方法实际上是启动了一个node进程，并且执行`script`指定的脚本，同时nodejs会在父子进程之间建立一个ipc双向通道，大致是可以理解为：
```javascript
function fork(scriopt, args, options) {
  const stdio = options.slient ? "pipe" : "inherit";
  
  return spawn("node", [script].concat(args), {
    stdio: [stdio, stdio, stdio, "ipc"]
  });
}
```

这就是本文的全部内容，以上~