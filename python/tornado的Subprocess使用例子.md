tornado是python的一个异步web框架，除了可以处理一般的web应用，tornado还可以处理很多异步的任务。今天要写的，是关于使用tornado的Subprocess类来处理子进程的调用的例子，这个demo来自于https://gist.github.com/FZambia/5756470
```python
    import subprocess
    import time

    from tornado.gen import Task, Return, coroutine
    from tornado.process import Subprocess
    from tornado.ioloop import IOLoop

    STREAM = Subprocess.STREAM
    PIPE = subprocess.PIPE

    # 执行子进程的核心函数
    @coroutine
    def call_subprocess(cmd, stdin_data = None, stdin_async = True):
        stdin = STREAM if stdin_async else PIPE
        # 创建子进程
        # 如果stdxx = STREAM，则意味着使用PipeIOStream作为输入输出流
        # 如果stdxx = PIPE，则意味着使用标准的进程输入输出管道缓冲区
        child = Subprocess(cmd, stdin = stdin, stdout = STREAM, stderr = STREAM)

        if stdin_data:
            if stdin_async:
                # tornado的Task，相当于js里的thunkify，作用是把带有callback的函数改写成返回Promise/Future
                yield Task(child.stdin.write, stdin_data)
            else
                child.stdin.write(stdin_data)

        if stdin_async or stdin_data
            child.stdin.close()

        # 等待结果，read_until_close，就是等待stdxx执行close的时候才返回
        result, error = yield [
            Task(child.stdout.read_until_close),
            Task(child.stderr.read_until_close)
        ]

        # cornado中协程的返回不能直接return，而是通过抛出异常来完成
        raise Return([result, error])

    def on_timeout():
        print('timeout')
        IOLoop.instance().stop()

    @coroutine
    def main():
        second_wait = 3
        deadline = time.time() + second_wait

        # 为io事件队列设置回调，类似js的setTimeout
        IOLoop.instance().add_timeout(deadline, on_timeout)

        result, error = yield call_subprocess(['svn', 'help'])
        print(result)

        IOLoop.instance().stop()

    if __name__ == '__main__':
        ioloop = IOLoop.instance()
        # 为io事件队列设置一个执行函数，类似js的nextTick
        ioloop.add_callback(main)
        ioloop.start()
```