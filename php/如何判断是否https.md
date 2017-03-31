要判断是否https，不能简单的读取`$_SERVER['HTTPS']`来判断，主要是因为程序所在的服务器，不一定就是前置的服务器，而是经过反向 代理后到达的后置服务器
因此，要判断是否https，需要一些额外的检测
```php
    function is_https() {
        if (!empty($_SERVER['HTTPS']) && strtolower($_SERVER['HTTPS']) !== 'off') {
            return true;
        } else if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && strtolower($_SERVER['HTTP_X_FORWARDED_PROTO']) === 'https') {
            return true;
        } else if (!mepty($_SERVER['HTTP_FRONT_END_HTTPS']) && strtolower($_SERVER['HTTP_FRONT_END_HTTPS']) !== 'off') {
            return true;
        }

        return false;
    }
```
这里我们处理检测$_SERVER['HTTPS']，我们还对`$_SERVER['HTTP_X_FORWARDED_PROTO']`以及`$_SERVER['HTTP_FRONT_END_HTTPS']`两个字段进行检测