在上一篇文章《http.createServer竟然背着我做了这些事情》中，我们已经实现了一个能完成基本http通信的乞丐版http服务器，但其中存在一些问题，包括：

* 这个基础版本的http服务器，只支持ipv4协议，而我们的http.createServer实际是同时支持ipv4和ipv6的
* 当我们完成一次请求后，关闭程序（ctrl-c)，并马上重启服务时，会出现 **Address already in use** 错误

而今天这篇文章，我们就来尝试修复这些问题，同时对整个程序的结构进行调整，方便之后的扩展。

## 同时支持ipv4和ipv6

### 分析代码中的那些与ipv4协议强关联的部分

要让我们的http服务同时支持ipv4和ipv6协议，首先我们就要找出代码中hard code的部分。这样的代码主要存在与以下两个地方：

1. 在调用`socket`函数时，传入的`AF_INET`参数，该参数在ipv6协议下，应该改为`AF_INET6`;
```c
/** 
 * ipv4
 * int listenfd = socket(AF_INET, SOCK_STREAM, 0);
 */
int listenfd = socket(AF_INET6, SOCK_STREAM, 0);
```

2. 在`bind`方法中，用于保存本地ip地址和端口的`struct sockaddr_in`结构体，在ipv6协议下，应该改为`struct sockaddr_in6`
```c
/**
 * ipv4
 * struct sockaddr_in servaddr;
 * memset(&servaddr, 0, sizeof(servaddr));
 * servaddr.sin_family = AF_INET;
 * servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
 * servaddr.sin_port = htons(8080);
 */
struct sockaddr_in6 servaddr;
memset(&servaddr, 0, sizeof(servaddr));
servaddr.sin6_family = AF_INET6;
servaddr.sin6_addr.s_addr = htonl(INADDR_ANY);
servaddr.sin6_port = htons(8080);
```

要去掉这些hard code的部分，能想到的最简单的方式，就是通过预定义宏来判断系统是否支持ipv6，并针对不同的情况使用不同的代码。然而这并不是最好的方法。接下来我就会介绍一个unix系统提供的方案，能非常优雅的解决这个对协议支持的问题。

### 通过`getaddrinfo`方法，创建协议无关的tcp服务

unix系统提供了一个`getaddrinfo`方法，主要用于dns域名解析的。我们只须向`getaddrinfo`方法中传入域名（如www.qq.com）和服务名称（如tcp），系统则会通过dns协议，查找到与该域名和服务对应的ip地址和端口列表，同时把与该域名和服务相关的，如cname，ip协议版本等信息一并返回。

除了能完成dns查询以外，`getaddrinfo`还能用于获取本地可用ip地址和端口的信息，如此我们就可以利用它的这个能力来实现一个协议无关的http服务。

接下来，我们首先需要编写一个名为`tcp_listen`的函数，该函数用于建立基础的tcp服务。在`tcp_listen`函数中，我们会调用`getaddrinfo`方法来获取本地可用ip地址和端口：

```c
#include <netdb.h>
#include <assert.h>

int tcp_listen(const char *host, const char *serv)
{
    // 获取可用ip地址和端口列表，结果以链表的形式返回，链表中的每一项为一个addrinfo结构体
    struct addrinfo hints, *res;
    memset(&hints, 0, sizeof(hints));
    hints.ai_flags = AI_PASSIVE;
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    getaddrinfo(host, serv, &hints, &res);

    // 遍历所有可用的ip地址和端口，逐一尝试建立tcp服务
    int fd;
    struct addrinfo *cur;

    for (cur = res; cur != NULL; cur = cur->ai_next)
    {
        if ((fd = socket(cur->ai_family, cur->ai_socktype, cur->ai_protocol)) < 0)
            continue;

        if (bind(fd, cur->ai_addr, cur->ai_addrlen) == 0)
            break;

        close(fd);
    }
    
    // 如果所有尝试均失败，则返回-1表示错误
    if (cur == NULL)
    {
        freeaddrinfo(res);
        return -1;
    }

    // 否则返回已创建服务的文件描述符
    freeaddrinfo(res);
    listen(fd, 512);

    return fd;
}
```

函数`tcp_listen`中主要完成了以下几个事情：

1. 调用`getaddrinfo`方法，并传入域名和服务名称，当需要获取本地可用ip地址时，域名传入`NULL`。注意，`getaddrinfo`方法接受一个提示参数（hints）,用于告诉系统我们需要获取的ip地址和端口类型，在这里我们传入了一个`addrinfo`结构体，其中的`ai_flags`字段设置为`AI_PASSIVE`，意味着我们要获取的地址信息将用于被动打开，亦即http服务端。而`ai_family`字段设置为`AF_UNSPEC`，意味着返回所有可以ip地址，包括ipv4和ipv6，最后`ai_socktype`字段设置为`SOCK_STREAM`，即返回的端口需要支持tcp协议。当`getaddrinfo`方法调用成功时，会返回一个链表，其中的每一项为一个`addrinfo`结构体。由于这个链表的内存是由系统分配的，因此在使用完后，需要调用`freeaddrinfo`方法释放内存。

2. 在获取可用ip地址和端口列表后，我们就可以遍历这个列表，逐一的尝试，直到找到一个可用的ip地址和端口后跳出循环，并返回对于的文件描述符。如果所有的ip地址和端口都无法建立其tcp服务时，则返回-1，告知调用方调用失败。

这时，我们就可以利用这个`tcp_listen`方法来代替原来的建立tcp服务的代码：

```c
int main()
{
    int listenfd = tcp_listen(NULL, "8080");

    for (;;)
    {
        int connfd = accept(listenfd, NULL, NULL);

        //...
    }

    return 0;
}
```

### 解决 **Address already in use** 错误

正如文章一开始提到，当我们完成一次请求后，关闭程序（ctrl-c)，并马上重启服务时，会出现 **Address already in use** 错误。其实这个错误主要是由于tcp连接其主动断开一端（通常为服务端）在关闭连接后会进入到time wait状态，并持续约75秒，以确认对端已完全断开连接。而在着75秒期间，之前连接所使用的ip地址和端口都会被占有。如果这时再次建立tcp服务，并尝试绑定到原ip地址和端口上时，系统就会提示 **Address already in use**。

为了解决这个问题，我们就需要通过设置`SO_REUSEADDR`这个socket选项，来告知系统允许对同一对ip地址和端口进行重复绑定，具体的代码如下:

```c
int tcp_listen(const char *host, const char *serv)
{
    // ...

    for (cur = res; cur != NULL; cur = cur->ai_next)
    {
        if ((fd = socket(cur->ai_family, cur->ai_socktype, cur->ai_protocol)) < 0)
            continue;

        int on = 1;
        setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));

        if (bind(fd, cur->ai_addr, cur->ai_addrlen) == 0)
            break;

        close(fd);
    }

    return fd;
}
```

在上面的代码中，我们在`listen`方法和`bind`方法之间调用了`setsockopt`方法，并指定开启`SO_REUSEADDR`选项。这样当我们再次重启服务时，就不会再收到 **Address already in use** 错误了。

## 结构调整

为了方便之后的扩展，这里我们对代码进行了一些简单的结构调整，抽象出`server_t`类和`session_t`类，分别用于管理tcp服务和单个tcp连接：

### `server_t`类，用于管理tcp服务

```c
typedef {
    int fd;
} server_t;

server_t *server_listen(const char *host, const char *serv)
{   
    server_t *srv = (server_t *)malloc(sizeof(server_t));
    assert(srv);

    srv->fd = tcp_listen(host, serv);

    return srv;
}

void server_close(server_t *srv)
{
    close(srv->fd);
    free(srv);
}
```

### `session_t`类，用于管理tcp连接

```c
typedef {
    int fd;
} session_t;

session_t *session_accept(server_t *srv)
{
    session_t *sess = (session_t *)malloc(sizeof(session_t));
    assert(sess);
    
    sess->fd = accept(srv->fd, NULL, NULL);

    return sess;
}

void session_close(session_t *sess)
{
    shutdown(sess->fd, SHUT_WR);
    free(sess);
}
```

### 实现http_handle函数，分离http协议的处理逻辑

```c
// 读取连接中的数据
void http_handle(session_t *sess)
{
    char buf[1024];
    int len = sizeof(buf) - 1;
    int nread;

    while ((nread = read(sess->fd, buf, len)) != 0)
    {
        // 如果read函数返回值小于0，则表示读取出现异常
        if (nread < 0)
        {
            // 如果异常是由软件中断引起的，则忽略，继续读取
            if (nread == EINTR)
                continue;
            // 否则，退出程序
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
    write(sess->fd, buf, strlen(buf));
}
```

经过上述的整理，现在我们的`main`函数就变得十分清晰了：

```c
int main()
{
    server_t *srv = server_listen(NULL, "8080");

    for (;;)
    {
        session_t *sess = session_accept(srv);
        
        http_handle(sess);

        session_close(sess);
    }

    server_close(srv);
}
```

以上～