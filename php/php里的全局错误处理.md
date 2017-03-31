在nodejs中，我们可以通过在`process`对象中监听`uncaughtException`事件来捕获那些在业务层没有捕获的错误
而在php中，其实也有类似的机制
```php
    set_error_handler(global_error_handler);
    set_exception_handler(global_exception_handler);
    register_shutdown_function(global_shutdown_handler);
```

我们首先搞清php中`error`和`exception`的区别。在php中，`exception`是所有可以修复的错误，也就是我们可以通过try-catch来捕获并修复这个错误的
而`error`则是那些不可修复的错误，或者是通过调用`trigger_error`方法触发的错误

在这里我们首先来列出一下，php有哪些类型的错误
```php
    E_ERROR             // X    致命的运行时错误。这类错误一般是不可恢复的情况，例如内存分配导致的问题。后果是导致脚本终止不再继续运行。
    E_WARNING           //      运行时警告 (非致命错误)。仅给出提示信息，但是脚本不会终止运行。
    E_PARSE             // X    编译时语法解析错误。解析错误仅仅由分析器产生。
    E_NOTICE            //      运行时通知。表示脚本遇到可能会表现为错误的情况，但是在可以正常运行的脚本里面也可能会有类似的通知。

    E_CORE_ERROR        // X    在PHP初始化启动过程中发生的致命错误。该错误类似 E_ERROR，但是是由PHP引擎核心产生的。
    E_CORE_WARNING      // X    PHP初始化启动过程中发生的警告 (非致命错误) 。类似 E_WARNING，但是是由PHP引擎核心产生的。

    E_COMPILE_ERROR     // X    致命编译时错误。类似E_ERROR, 但是是由Zend脚本引擎产生的
    E_COMPILE_WARNING   // X    编译时警告 (非致命错误)。类似 E_WARNING，但是是由Zend脚本引擎产生的。

    E_USER_ERROR        //      用户产生的错误信息。类似 E_ERROR, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。
    E_USER_WARNING      //      用户产生的警告信息。类似 E_WARNING, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。
    E_USER_NOTICE       //      用户产生的通知信息。类似 E_NOTICE, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。

    E_STRICT            //      启用 PHP 对代码的修改建议，以确保代码具有最佳的互操作性和向前兼容性。
    E_RECOVERABLE_ERROR //      可被捕捉的致命错误。 它表示发生了一个可能非常危险的错误，但是还没有导致PHP引擎处于不稳定的状态。 如果该错误没有被用户自定义句柄捕获 (参见 set_error_handler())，将成为一个 E_ERROR　从而脚本会终止运行。

    E_DEPERECATED       //      运行时通知。启用后将会对在未来版本中可能无法正常工作的代码给出警告。
    E_USER_DEPRECATED   //      用户产少的警告信息。 类似 E_DEPRECATED, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。

    E_ALL               //      E_STRICT出外的所有错误和警告信息。
```

所有从php代码抛出的`exception`，如果没有被try-catch捕获，都会去到`global_exception_handler`
如果用户没有设置`global_exception_handler`，那么php就会出现一个Fatal error错误，该错误为一个`E_ERROR`类型的错误。

而php出现的`error`，则除了上面打X的那些类型以外，都会去到`global_error_handler`，在这个`global_error_handler`中，由于已经不走php标准的错误处理流程，我们有需要调用`exit`来让php终止执行

而在php出现错误未被处理时，php就会终止执行，然后调用`global_shutdown_handler`，我们可以通过调用`error_get_last`方法来获取错误信息

这里有个小技巧，由于上面被打X的错误类型，都不会经过`global_error_handler`，但会经过`global_shutdown_handler`，因此可以通过在`global_shutdown_handler`中捕获这些类型的错误，并交给`global_error_handler`统一处理。