在php中，我们可以通过常量`PHP_VERSION`来获取当前php的版本，但这个版本信息并非纯数字组成，而是会带有一些系统信息，例如`5.5.9-1ubuntu4.17`
这是，我们需要判断当前php版本是否大于某个版本，就可以借助`version_compare`函数来完成
```php
    function is_php($ver) {
        static $is_php = [];

        if (!array_key_exists($ver, $is_php)) {
            $is_php[$ver] = version_compare(PHP_VERSION, $ver, '>=');
        }

        return $is_php[$ver];
    }
```
这里使用了一个静态关联数组来缓存之前的结果，这样就不会每次都去计算，然后通过`version_compare`方法，来比较两个版本号