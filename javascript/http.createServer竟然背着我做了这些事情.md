前些天和客户端的同事聊天，说起了关于在node中创建服务器相关的话题。其中说到，如果要在node中要创建一个服务器，只需要简单的一两行代码即可：

```javascript
const http = require('http')

const server = http.createServer((req, res) => {
    // 业务逻辑
})

server.listen(8080)
```

客户端的同事听到后羡慕不已。因为要系统层面从零开始实现一个支持io多路复用的服务器，少则页需要一百多行的代码。当时，我的心里是笑出猪声的。庆幸自己在入行的时候选了前端这个行当，选了js这门语言。

不过很快，我就意识到，这种情况并没有什么值得高兴的，甚至，我应该为此感到悲哀，我竟然到`http.createServer`这句代码一无所知。求生欲顽强的我有点坐不住了，趁着最近的长假翻了一些关于系统编程和网络编程的书籍，也读了一些开源项目的源码，心里才安稳了下来，同时也对`http.createServer`背后默默完成的工作有了一些了解。在此记录一下我的收获。

## 从零开始，实现一个支持io多路复用的http服务器

为了能更完整的展示出`http.createServer`的底层实现，我打算从零开始实现一个支持io多路复用的http服务器。注意，阅读以下代码，需要有一定的c语言基础，以及对unix系统调用的基本了解。而为了简单起见，我会省略部分错误处理，以使代码的意图更加清晰。

## 第一步，创建一个基础的tcp服务

作为前端工程师，我们或多或少都有听说过tcp/ip协议族，也知道http协议是一个基于tcp协议之上的应用层协议。因此为了实现http服务，首先我们地从tcp服务开始。
> 这时可能就会有同学会问，tcp协议是建立在ip协议之上的啊，为什么不从ip协议，甚至更底层的以太网协议开始呢？其实这并不是随意选择的结果，而是由操作系统提供的接口（系统调用）决定的。在unix操作系统开发之初，其开发人员就发现，通常情况下，传输层协议及以下其他层级的协议实现在实际开发中是极少改动的，因此他们选择了以传输层作为分界，传输层及以下层级的协议由系统实现，而应用层协议则由应用开发者自己实现。

以下为具体的创建tcp服务的代码实现：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main()
{
    // 创建支持tcp，ip协议栈的socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);

    // 把第一步中创建的socket，绑定到本地ip地址和端口上
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(8080);
    
    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

    // 把第一步中创建的socket，从主动打开状态转为被动打开状态
    listen(listenfd, 512);

    // 其他代码

    return 0;
}
```
要建立一个基础的tcp服务，需要三步：

1. 首先我们通过`socket`函数，创建一个socket，并返回其文件描述符。其中的`AF_INET`参数说明我们的socket需要支持ipv4协议，而`SOCK_STREAM`参数，说明我们的socket支持字节流，亦即是支持tcp协议。

2. 然后我们创建了一个`sockaddr_in`结构体，该结构体用于保存本地ip地址以及端口信息。这里我们通过设置`sin_family`字段为`AF_INET`，指定该ip地址为ipv4地址。然后我们把`INADDR_ANY`这个ipv4地址写入到`sin_addr`字段中，表示接受来自本地任意网卡设备上ip数据包。同时我们也在`sin_port`字段中写入`8080`，这个比较好理解，就是指接受来自`8080`端口的tcp字节流。最后我们通过`bind`函数把在第一步中创建的socket与上述的本地ip地址和端口绑定到一起。

3. 最后我们调用了`listen`函数，这个函数名称有点令人迷惑，实际上它的作用是把socket转换为被动打开状态，也就是被动的接受来自客户端的连接请求（所有在unix系统上创建的socket，默认进入主动打开状态）。其中的参数`512`用于指定“已连接队列”的长度（当连接请求来到服务器并完成三次握手后，系统就会把该连接放入“已连接队列”中等待处理，一旦处于等待的连接数超过队列的长度，就会直接丢弃）。

### 第二步，获取tcp连接，读取请求，并写入响应

在上一步中，我们已经建立起一个基础的tcp服务，接下来，我们就要等待来自客户端的连接请求。一旦客户端发起连接，并完成三次握手，我们就可以开始对该连接进行读写，以下为具体的代码实现：

```c
int main()
{
    // ...

    for (;;)
    {
        // 获取“已连接队列”中的第一个连接
        int connfd = accept(listenfd, NULL, NULL);

        // 读取连接中的数据
        char buf[1024];
        int len = sizeof(buf) - 1;
        int nread;
        while ((nread = read(connfd, buf, len)) != 0)
        {
            // 如果read函数返回值小于0，则表示读取出现异常
            if (nread < 0)
            {
                // 如果异常是由软件中断引起的，则忽略，继续读取
                if (nread == EINTR)
                    continue;
                // 否则，退出请求
                perror("read");
                exit(1);
            }

            // 把读取到的数据输出到标准输出
            buf[nread] = 0;
            fputs(buf, stdout);

            // 如果读取到的数据以\r\n\r\n结束，则认为请求结束
            // 这是由http协议定义的结束标志
            if (!strncmp(buf + nread - 4, "\r\n\r\n", 4))
            {
                break;
            }
        }

        // 把需要回复的数据组装成http响应，写入连接
        strcpy(buf, "HTTP/1.1 200 OK\r\n\r\nHello World\r\n\r\n");
        write(connfd, buf, strlen(buf));

        // 最后关闭连接
        close(connfd);
    }

    return 0;
}
```

以上的代码主要完成了这样几个事情：

1. 通过`accept`函数，我们从系统的“已连接队列”中获取到第一个等待处理的连接，这里值得关注的是，如果当前“已连接队列”为空，则`accept`函数会一直处于阻塞状态，直到新的连接到来。一旦获取到等待处理的连接，`accept`函数就会返回一个代表该连接的文件描述符。

2. 接下来我们就开始从连接中读取数据。由于不知道连接着具体包含多数数据，这里我们预先准备了一个大小为1024个字节的缓冲区，通过循环调用`read`函数，不断的从连接着读取数据。同时，我们把读取到的数据，写到标准输出中，方便调试。此外，在http协议中规定，请求和响应均以两个空行（`\r\n\r\n`）为结束标志，因此在循环读取数据的过程中，我们需要根据该标志来判断请求的结束（这里是简单处理，实际的情况会复杂得多），一旦请求结束，就需要跳出循环。

3. 在完整的读取到请求后，我们就可以根据请求中的内容来构造对应的响应。这里由于我们的目的只是了解`http.createServer`背后的原理，因此对于响应的内容简单处理，返回一个友好的`Hello World`。注意这里并不是简单的写入`Hello World`，而是要根据http协议构造一个完整的响应，其中了http的版本信息以及响应状态码等信息。

4. 当响应发送完成后，服务器就会断开连接，并进入下一次的循环。

### 总结

今天给大家结束了在unix系统上如何建立起一个基础的tcp服务，并且在这个tcp服务上，获取到http请求，并且作出响应。现在我们就可以利用gcc来编译上面的代码（假设的代码保存在main.c文件中），编译完成后，就可以运行我们的程序：

```shell
$ gcc main.c -o server
$ ./server
```

这时我们就可以尝试通过使用`curl`这样的客户端程序来向我们的服务器发起请求，并观察其响应：

```shell
$ curl 127.0.0.1:8080
Hello World
```

在接下来的文章中，我会继续改进我们这个乞丐版的http服务，以上～