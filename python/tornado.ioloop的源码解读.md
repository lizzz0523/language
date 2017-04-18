为了更深入的了解tornado实现异步io的原理，花了点时间来看ioloop的源代码，其实可以说只要看懂了ioloop和iostream这两个包，基本对tornado的异步io原理就会有一个不错的了解，之后我也会有一篇关于iostream的源码解读文章。

要了解ioloop，我们不得不说一个老生常谈的话题，就是关于在unix上进行网络编程时会遇到的四个名词，阻塞、非阻塞、同步、异步。所以首先我们来说明以下这四种模式分别是什么，他们之间有什么联系和区别。

阻塞和非阻塞，主要是关于进程在调用方法后的不同状态
* 阻塞：进程在调用方法后，系统就会把该进程挂起，直到系统得到调用的结构后，才重新唤醒该进程
* 非阻塞：进程在调用方法后，系统会立即返回，而调用的结果是通过状态或者通知等形式获得

而同步和异步，则是关于进程在调用方法后，获取结果的不同方式
* 同步：进程在调用方法后，需要一直等待调用结果，在得到调用结果后，才能执行下一步的逻辑
* 异步：进程在调用方法后，不需要等待调用结果，马上可以执行下一步逻辑，调用结构则是通过回调的形式得到

```python
    # 阻塞，同步
    result = read(fd)
    print result
    print 'end'

    # 非阻塞，同步
    while True:
        if result = read(fd):
            break;

    print result
    print 'end'

    # 非阻塞，异步
    read(fd, callback = lambda result: print result)
    print 'end'
```

正常来说，阻塞调用对于系统来说是最合理的，因为当结果还没有准备好时，系统先让进程挂起，这样能最大限度的节省cpu，但这样做对io来说却是十分糟糕的，同一个进程，一次只能处理一个io，对于高并发的web应用来说，则需要通过多进程或者是多线程的方式来处理多个请求，系统开销十分大。

为了解决这个问题，提高阻塞调用的io效率，就需要引入一个叫io多路复用的东西，说白了就是一个代理中介，在这个代理上我们可以同时监听多个io的状态，一旦某个io有数据返回，则唤醒应用进程，这样即使是调用是阻塞的，但由于每个io是独立处理的，一旦某个io数据有返回，应用进程就马上被唤醒，触发之前注册的与该io相对应的回调函数，达到了类似异步的效果。

在linux上，这样的代理有几种，分别是select，poll，epoll，kqueue...，他们各有优缺点，其中epoll，kqueue的性能是最好的，这是由于select，poll把所有io的状态都返回给应用进程，应用进程需要通过轮询所有的io状态，来判断需要处理哪些io，在同时监督多个io时，其时间复杂度为O(n)，n为监听io的数量。

而epoll，kqueue则是针对每个io分别设置回调，只有状态发生变化的io才会返回给应用进程，应用进程不需再去轮询，其时间复杂度为O(1)，因此能轻松面对c10k问题。

除了io多路复用，在系统层面其实还有其他方式能实现异步，他们分别是信号驱动的io和异步io。但信号驱动的io只能处理udp协议的io，对tcp协议的则无法使用（主要是由于在tcp协议下，能触发信号的情况太多，而应用进程无法区分不同的情况，而在udp协议下，触发信号的情况只有两种，一种是数据到达，一种是出现错误，因此应用进程就能根据是否存在错误来处理信号）。

对于异步io，linux下的aio_*类函数则暂时没被广泛使用（在linux内核2.6之后才正式纳入其标准）。

在此我们先总结一下在，在unix下的网络编程，主要会涉及一下五种模式：
* 阻塞io
* 非阻塞io
* io多路复用
* 信号驱动io
* 异步io

[unix网络编程](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1492497158399&di=5452400ca1d3af8d5ebf90c74b8d78be&imgtype=0&src=http%3A%2F%2Fwww.cfanz.cn%2Fuploads%2Fpng%2F2015%2F12%2F16%2F19%2FX5fY963245.png)

说了这么多，其实主要是想说明，tornado实现异步io，采用的是io多路复用的方法，也就是通过调用系统的select，poll，epoll，kqueue来实现异步io。而tornado.ioloop.IOLoop则是对这些代理进行封装，这些我们可以在IOLoop类的注释中得到证实
```python
    class IOLoop(Configurable):
        """A level-triggered I/O loop.
        We use ``epoll`` (Linux) or ``kqueue`` (BSD and Mac OS X) if they
        are available, or else we fall back on select(). If you are
        implementing a system that needs to handle thousands of
        simultaneous connections, you should use a system that supports
        either ``epoll`` or ``kqueue``.
        “”“
```