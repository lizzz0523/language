之前刷知乎，看到有个哥们分享了他在面试头条前端岗时的碰到的题目，其中有这么一道题：
> `funtion request(urls, maxNumber, callback)`，要求编写函数实现，根据`urls`数组内的url地址进行并发网络请求，最大并发数`maxNumber`，当所有请求完毕后调用`callback`函数（已知请求网络的方法可以使用fetch api）

刚好周末比较闲，就试着写了一下。在贴代码之前，还是先说说题目吧，其实根据我自己的经验，在前端页面控制请求并发数的情况其实并不多见，一个页面同时发起的请求数一般都在个位数，而且浏览器对于同一域名也有并发控制，在我的记忆里，只有之前在微云的分片上传功能上，使用过并发控制。所以就题目本身而言，要实现这样的能力并不“实用”。

虽然控制并发请求数在前端页面不常见，但并发控制本身还是很有价值的。例如在日志文件的写入队列，爬虫的并发控制，golang中的goroutines调度等等都会使用到并发控制。因此我这次着重编写的实际是并发控制的代码，对于请求的并发控制可以理解为其中一种应用场景。当然我这里只是简单的实现，可能有些细节并未考虑。

并发控制，其实在生活中也很常见。你可以想象一下去银行办理业务：
1. 首先你会到取号机获取一个流水号，然后就会到等候室等待被叫
2. 银行会同时开4，5个业务窗口，每个窗口在处理完前一个业务后，就会根据流水号呼叫下一位客户
3. 直到所有的客户业务都办理完毕，银行就会结束当天的工作

这个就是一个非常简单的并发控制模型，在这个模型里，有两个重要的组成部分：
1. 等候室
2. 业务窗口

如果用编程的方式来说明的话，等候室就是任务队列，业务窗口就是任务处理器。只要完成这两个部分抽像，就能完成基本的并发控制。接下来我首先完成任务队列的部分，在js中，队列只需要使用一个数组就可以实现，而对于“任务”，我选择使用一个`Task`类对其进行抽象：

```javascript
class Task {
  constructor (run, ...args) {
    this.run_ = run
    this.args_ = args
  }

  async run () {
    await this.run_(...this.args_)
  }
}
```

这里我把需要运行的函数`run`，和`run`函数使用到的参数`args`传入到`Task`类中，这样我们只需要执行`Task::run`方法，就可以处理不同的任务了。

而对于任务处理器，其功能就是从任务队列中获取任务，每次获取一个任务，执行完毕后再获取下一个，直到所有任务都完成为止，我选择使用一个`TaskRunner`类对其进行抽象：

```javascript
class TaskRunner {
  constructor (tasks) {
    this.tasks_ = tasks
  }

  async run () {
    while (this.tasks_.length) {
      const task = this.tasks_.shift()
      await task.run()
    }
  }
}
```

这里所有处理器共享同一个任务队列，因此任务队列会作为参数传入到`TaskRunner`类的构造函数中，`TaskRunner::run`方法则是逐次从任务队列中取出任务并执行，直到所有的任务都处理完。

哈哈，是不是很简单。最后我们实现一个`run`方法，来传入任务和最大并发数：

```javascript
async function run (tasks, max) {
  const wait = []
  for (let i = 0; i < max; i++) {
    const runner = new TaskRunner(tasks)
    wait.push(runner.run())
  }
  return Promise.all(wait)
}
```

至此，我们已经完成这个基本的并发控制模型了，接下来顺手把题目中的`request`函数实现一下：

```javascript
function request (urls, maxNumber, callback) {
  const tasks = urls.map(url => new Task(fetch, url))
  run(tasks, maxNumber).then(callback)
}
```

ok，感觉有些简单啊，欲求不满啊。为了练练手，下面再给出一个过程式的实现，过程式的就不需要什么类了，直接实现一个`run`函数即可，其实讲真的，过程式的实现看起来还是没有面向对象的清晰，有兴趣的小伙伴，也可以给出自己的实现哦。

```javascript
async function run (tasks, max = 4) {
  return new Promise(resolve => {
    let running = 0
    next()

    function next () {
      if (running === 0 && tasks.length === 0) {
        resolve()
      } else {
        while (running < max && tasks.length) {
          ;(async () => {
            running++
            const task = tasks.shift()
            await task()
            running--
            next()
          })();
        }
      }
    }
  })
}

function request (urls, maxNumber, callback) {
    const tasks = urls.map(url => fetch.bind(null, url))
    run(tasks, maxNumber).then(callback)
}
```

以上～～