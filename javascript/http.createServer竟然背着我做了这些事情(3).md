在上一篇文章[《http.createServer竟然背着我做了这些事情(2)》](https://github.com/lizzz0523/language/blob/master/javascript/http.createServer%E7%AB%9F%E7%84%B6%E8%83%8C%E7%9D%80%E6%88%91%E5%81%9A%E4%BA%86%E8%BF%99%E4%BA%9B%E4%BA%8B%E6%83%85(2).md)中，我们在能完成基本tcp通信的乞丐版http服务器之上，解决了以下两个问题：

* 使我们的服务器同时支持ipv4和ipv6
* 允许服务器重复绑定本地端口，解决 **Address already in use** 错误

而今天这篇文章，我们就集中精力的解决一个核心问题--**http协议解析**。

要完整的介绍http协议已经超出了本篇文章的范围，可以写成一本完整书。因此我们这里只是简单的介绍一下，以下文字摘录于[维基百科](https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)：

> http协议，中文叫做超文本传输协议，是一种用于分布式、协作式和超媒体信息系统的应用层协议。最初设计的目的是为了提供一种发布和接收HTML页面的方法。

从过程来分，http协议可以分成两个部分，一个是来自客户端发起http请求，另一个是来自服务端返回的响应。

而从内容来分，http协议则可以分成三个部分，一个是http请求/响应行，然后是http头部，最后是http消息体，而http请求和http响应在内容上的主要分别在于请求/响应行的格式不同，下面是一个简单的例子：

http请求
```
GET / HTTP/1.1
Host: www.qq.com
```

http响应
```
HTTP/1.1 200 OK
Date: Sat, 13 Oct 2018 16:41:36 GMT
Content-Type: text/html; charset=GB2312
Transfer-Encoding: chunked
Connection: keep-alive
Server: squid/3.5.24
Vary: Accept-Encoding
Expires: Sat, 13 Oct 2018 16:42:36 GMT
Cache-Control: max-age=60
Content-Encoding: gzip

<!DOCTYPE html>
<html lang="zh-CN">
    <head>
        <title>腾讯首页</title>
    </head>
    </body>
        ...
    </body>
</html>
```

从以上的这个例子中我们可以看到http协议是一个明文协议，也就是说，在tcp连接上传输的所有http消息，均以非加密文本的形式出现。这些http消息是由多个行组成，行与行之间通过换行符`"\r\n"`分割。

每次传输的http消息中的第一行就是http请求/响应行，当中包含了请求方法（`GET`），请求路径（`/`），http版本号（`HTTP/1.1`），响应状态码（`200`）以及描述响应状态的短语（`OK`）等信息。

在请求/响应行的下一行开始就是http头部，每一行都是一对key-value值（`Host: www.qq.com`），key和value中间通过`: `分割。

而http消息体则不是必须的，如果存在http消息体，则http头部和http消息体之间会通过两次换行符（`\r\n\r\n`）分割。http消息体中可以包含多种格式的数据（最常见的是html），直到遇到另一个两次换行符（`\r\n\r\n`），标记着本次http消息结束。

这里只是对http协议的简单介绍，真实情况下的http请求和响应要比上面说到的负责得多。虽然如此，但要在我们的http服务中解析http协议则并没没有想象中困难，原因是nodejs的开发团队已经把在nodejs中使用到的，c语言版本的[http parser库](https://github.com/nodejs/http-parser)独立开源了。我们只需使用他就能完成http协议的解析。

要使用这个http-parser库，我们首先要把他从github上克隆到我们的项目文件夹下（假设我们的项目文件夹叫http-server）：

```shell
$ cd http-server
$ git clone https://github.com/nodejs/http-parser.git
```

然后我们就可以开始在我们的代码中使用http-parser了。http-parser对外提供了一些基于回调函数的接口，当解析到http消息的某个部分时，就会触发对应的回调函数，并且传入解析的结果：

```c
int my_url_callback(http_parser* parser, const char *url, size_t len) {
    // 向标准输出打印url
    fwrite(url, len, 1, stdout);
    return 0;
}

// 创建http_parser_settings实例，并写入my_url_callback回调函数
http_parser_settings settings;
settings.on_url = my_url_callback;

// 创建http_parser实例，用于解析http协议
http_parser *parser = malloc(sizeof(http_parser));
http_parser_init(parser, HTTP_REQUEST);

// 解析http协议，其中buf为包含http消息的内存缓冲区，len为buf的长度
http_parser_execute(parser, &settings, buf, len);
```

上面是请求修改自[README](https://github.com/nodejs/http-parser/blob/master/README.md)中的一段demo，用于解析http请求中的请求路径。

有了这些准备后，我们就可以正式向我们的http服务器中添加解析http协议的相关代码了。首先我们定义一个`request_t`类，用于保存解析http请求中的相关数据，其中包括了用于保存http-parser和其相关设置的`settings`和`parser`字段，还用用于保存http请求信息的`http_version`，`http_method`，`http_url`，`http_headers`，`http_body`等字段，其中的`http_headers`我们使用了一个自定义的哈希表`hashmap_t`实例来承载。最后还有一个`is_ready`字段用于保存当前http消息时候已经完整：

```c
typedef struct {
    http_parser_settings *settings;
    http_parser *parser;
    hashmap_t *http_headers;
    char *http_version;
    char *http_method;
    char *http_body;
    char *http_url;
    int is_ready;
} request_t;
```

接下来，就要定义一些与这个`request_t`类的创建和销毁相关的方法：

```c
static int on_message_complete(http_parser *parser);
static int on_headers_complete(http_parser *parser);
static int on_url(http_parser *parser, const char *str, size_t len);
static int on_header_field(http_parser *parser, const char *str, size_t len);
static int on_header_value(http_parser *parser, const char *str, size_t len);
static int on_body(http_parser *parser, const char *str, size_t len);

// 初始化reqeust_t实例
static request_t *request_init()
{
    request_t *req = (request_t *)malloc(sizeof(request_t));
    assert(req);

    // 初始化settings，并写入各阶段的回调函数
    http_parser_settings *settings = (http_parser_settings *)malloc(sizeof(http_parser_settings));
    assert(settings);

    http_parser_settings_init(settings);
    settings->on_url = on_url;
    settings->on_header_field = on_header_field;
    settings->on_header_value = on_header_value;
    settings->on_body = on_body;
    settings->on_headers_complete = on_headers_complete;
    settings->on_message_complete = on_message_complete;

    // 初始化parser，并指定是用于解析http请求
    http_parser *parser = (http_parser *)malloc(sizeof(http_parser));
    assert(parser);

    http_parser_init(parser, HTTP_REQUEST);
    parser->data = req;

    req->settings = settings;
    req->parser = parser;

    req->http_headers = hashmap_init();
    req->http_version = NULL;
    req->http_method = NULL;
    req->http_body = NULL;
    req->http_url = NULL;
    req->is_ready = 0;

    return req;
}

// 释放reqeust_t实例
static void request_free(request_t *req)
{
    hashmap_free(req->http_headers);
    free(req->http_version);
    free(req->http_method);
    free(req->http_body);
    free(req->http_url);
    free(req->settings);
    free(req->parser);
    free(req);
}
```

然后，我们再来定义一些setter用于更新`request_t`实例数据：

```c
#define HTTP_SETTER(name)                                                  \
    do {                                                                   \
        if (req->http_##name == NULL)                                      \
            req->http_##name = (char *)malloc(len + 1);                    \
        else if (sizeof(req->http_##name) < len + 1)                       \
            req->http_##name = (char *)realloc(req->http_##name, len + 1); \
        assert(req->http_##name);                                          \
        memcpy(req->http_##name, str, len);                                \
        req->http_##name[len] = 0;                                         \
    } while (0)                                                            \

// 保存http请求url
static void request_set_url(request_t *req, const char *str, size_t len)
{
    HTTP_SETTER(url);
}

// 保存http请求消息体
static void request_set_body(request_t *req, const char *str, size_t len)
{
    HTTP_SETTER(body);
}

// 保存http请求方法
static void request_set_method(request_t *req, unsigned int method)
{
    const char *str = http_method_str(method);
    size_t len = strlen(str);

    HTTP_SETTER(method);
}

// 保存http当前版本号
static void request_set_version(request_t *req, unsigned short major, unsigned short minor)
{
    char str[20];
    snprintf(str, sizeof(str), "HTTP/%d.%d", major, minor);
    size_t len = strlen(str);

    HTTP_SETTER(version);
}

// 辅助函数，用于把字符串转换成小写字母
static char *lowercase(char *str)
{
    size_t len = strlen(str);
    int i;

    for (i = 0; i < len; i++)
        if (str[i] >= 'A' && str[i] <= 'Z')
            str[i] |= 0x20;

    return str;
}

// 获取http请求头，这里由于http-parser并没有把请求头中的key-value对一次返回
// 而是分成两次，先后通过不同的回调函数返回，因此这里通过使用一个静态变量key来处理key-value的对应关系
static void request_set_headers(request_t *req, const char *str, size_t len)
{
    static char *key = NULL;
    char *value;

    // 如果返回的是key，则把key转换成小写，并写入静态变量等到处理
    if (key == NULL)
    {
        key = strndup(str, len);
        key = lowercase(key);
        return;
    }
    
    // 如果返回的是value，连同key一并写入哈希表
    value = strndup(str, len);
    hashmap_insert(req->http_headers, key, value);

    free(key);
    free(value);

    key = NULL;
}
```

最后定义三个辅助函数，分别用于解析http请求以及构造http响应和错误响应，其中在构造http响应时，使用到自定义的buffer_t，用于处理变长字符串：

```c
static int http_request(session_t *sess, request_t *req)
{
    size_t nread;
    size_t nparsed;

    // 从连接中循环读取数据
    while ((nread = read(sess->fd, str, MAXLINE)) != 0)
    {
        if (nread < 0)
        {
            if (nread == EINTR)
                continue;
            return -1;
        }

        // 对读取到的数据进行http解析
        nparsed = http_parser_execute(req->parser, req->settings, str, nread);

        // 如果解析的过程出错，退出并告知调用方
        if (nparsed != nread)
            return -1;

        // 如果http请求已经接受完成，退出循环
        if (req->is_ready)
            break;
    }

    return 0;
}

// 构造http响应，此时的http响应状态码为200
static int http_response(session_t *sess, request_t *req)
{
    request_print(req);

    buffer_t *buf = buffer_init();
    buffer_append_string(buf, req->http_version);
    buffer_append_string(buf, " 200 OK\r\n\r\n");

    buffer_write(buf, sess->fd);

    buffer_free(buf);

    return 0;
}

// 构造http错误响应，此时的http响应状态码为500
static int http_error(session_t *sess, request_t *req)
{
    buffer_t *buf = buffer_init();
    buffer_append_string(buf, req->http_version);
    buffer_append_string(buf, " 500 INTERNAL_SERVER_ERROR\r\n\r\n");
    
    buffer_write(buf, sess->fd);

    buffer_free(buf);

    return 0;
}
```

有了上面这些函数，我们就可以清晰的完成整个http解析处理的过程了：

```c
int http_handle(session_t *sess)
{
    request_t *req = request_init();

    if (http_request(sess, req) < 0)
        goto error;

    if (http_response(sess, req) < 0)
        goto error;

    request_free(req);

    return 0;

error:
    http_error(sess, req);
    request_free(req);

    return -1;
}
```

以上～