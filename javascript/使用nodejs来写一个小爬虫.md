有时候我们需要从其他网站中获取一些有用的数据，当这些网站并没有提供相应的open api，我们只好通过爬取的方式来获得。一般我们写这些网络爬虫都会使用python，python也有类似scrapy这样的工具方便我们编写爬虫。但对于一些前端的同学来说，python不一定写得溜。不过，其实用nodejs来编写简单的网络爬虫并不困难，这里记录一下最近写的一个小爬虫，用于爬取coursera上的课程链接，当然如果要爬取课程相关的介绍，只要简单修改一下就可以。

写爬虫，最核心的要有两样工具，一个是发送网络请求，另一个是解析请求结果（html、json，image等）然后在结果里获取有用的内容。

在nodejs中，要发送请求，可以直接使用`http/https`模块中的相关方法来完成，但这些方法用起来还是比较晦涩，我们也可以使用像`request`这样的第三方模块来简化我们的工作，在我编写的爬虫里，使用的是`request-promise`，是对`request`的`promise`封装。

而在nodejs中，要解析json数据，我们直接使用原生的JSON.parse方法即可。但如果要解析的是html，则需要一个html parser来帮忙我们，在这里推荐一下`cheerio`，他提供了我们前端所熟悉的jquery-like接口，通过像`$('li').text()`这样的方式来获取我们所需的数据，非常的方便。

有了这两个工具，我们就可以开始写爬虫了。首先爬虫需要做的第一个事情是请求目标链接，并抓取我们感兴趣的内容。说起来很简单，但这里要注意，一般我们并不会直接打开目标链接，而是从某个入口开始进入，然后像平时浏览网站一样，一层一层的打开页面，直到我们感兴趣的目标页面。所以我们需要一个队列来记录那些需要访问的链接。

```javascript
    let request = require('request-promise');
    let cheerio = require('cheerio');
    
    const HOST_URL = 'https://www.coursera.org';
    
    class Spider {
        constructor(queue, record) {
            this.queue = queue;
            this.record = record;
            this.run_();
            
            // 由于有些网站对连续的请求会进行拦截，一般需要错开每次的请求，这里设置一个延迟时间
            this.delay = 100;
            this.timer = null;

            // 重试记录
            this.retry = {};
        }

        run_() {
            if (this.timer) {
                return;
            }

            let run = () => {
                this.timer = null;
                
                // 如果队列非空，则获取对首，并发起请求
                if (this.queue.length) {
                    this.get_(this.queue.shift());
                }

                if (this.queue.length) {
                    this.timer = setTimeout(run, this.delay);
                }
            }

            run();
        }

        get_(url) {
            console.log('start\trequest ' + url);
                
            request({
                uri: url,
                // 解析结果
                transform: (body) => {
                    return body[0] === '{' ? JSON.parse(body) : cheerio.load(body);
                },
                // 加入常见请求头
                headers: {
                    'Accept-Encoding': 'gzip, deflate, sdch, br',
                    'Accept-Language': 'zh-CN,zh;q=0.8,en;q=0.6,fr;q=0.4,ja;q=0.2,ru;q=0.2,zh-TW;q=0.2,ko;q=0.2',
                    'Upgrade-Insecure-Requests': '1', 
                    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36',
                    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                    'Cache-Control': 'max-age=0',
                    'Connection': 'keep-alive',
                }
            })
            .then((res) => {
                console.log('end\trequest ' + url);

                this.parse_(res, url);
                this.run_();
            })
            .catch((err) => {
                this.error_(err, url);
            });
        }

        error_(err, url) {
            console.log(err.message, url);

            // 对失败的链接进行重试
            if (url in this.retry) {
                return;
            }
            this.retry[url] = true;
            
            this.push_(url);
            this.run_();
        }

        parse_(res, url) {
            url = url.replace(HOST_URL, '');
            url = url.replace(/(^\/+|\/+$)/g, '');

            let len = url.split('/').length;

            switch (len) {
                case 1:
                this.parseDomainNav_(res);
                break;
                case 2:
                this.parseSubdomainSampler_(res);
                break;
                case 3:
                this.parseOfferingCard_(res);
                break;
            }
        }
        
        // 解析主导航上的链接，进入大类
        parseDomainNav_($) {
            this.push_(
                $('.rc-DomainNav a[data-track-component="browse_left_nav"]')
                .map((index, elem) => $(elem).attr('href'))
                .get()
            );
        }
        
        // 解析大类里的查看全部按钮，进入小类
        parseSubdomainSampler_($) {
            this.push_(
                $('.rc-SubdomainSampler a[data-track-component="subdomain_see_all"]')
                .map((index, elem) => $(elem).attr('href'))
                .get()
            );
        }
        
        // 解析小类里的课程链接，并保存
        parseOfferingCard_($) {
            this.push_(
                $('a[data-track-component="offering_card"]')
                .map((index, elem) => $(elem).attr('href'))
                .get()
            );
        }

        push_(urls) {
            urls.forEach((url) => {
                url = HOST_URL + url;
                
                // 判断是否我们感兴趣的页面
                if (url.match(/^\/browse/)) {
                    // 把url方法待访问列表中
                    this.queue.push(url);
                } else {
                    // 把url写入文件
                    this.record.push(url);
                }
            });
        }
    }
```

有了一个能爬取数据的爬虫，我们还需要一个文件写入的功能，把爬取到的数据保存下来，当然你也可以直接保存到数据库中。写入文件的时候，需要注意的是写入冲突的问题，因为我们并不知道数据什么时候需要写入，有可能在前一次写入没完成之前，就开始了另一次写入，这样就会产生冲突。解决这类问题只要加入一个缓冲队列即可。在写入数据之前，先检查一下当前是否正在写入，如果是，先把数据写入缓存队列，如果不是，这直接写入文件。

```javascript
    let path = require('path');
    let fs = require('fs');
    
    function fwrite(options) {
        return new Promise((resolve, reject) => {
            let { file, data } = options;

            fs.appendFile(file, data, (err) => {
                if (err) {
                    throw err;
                }

                resolve();
            });
        });
    }

    class Record {
        constructor() {
            this.queue = [];
            // 记录是否正在写入
            this.isFlushing = false;
        }

        push(urls) {
            if (!Array.isArray(urls)) {
                urls = [urls];
            }
            
            if (this.isFlushing) {
                // 如果正在写入文件，则先把数据写入缓存队列
                this.push_(urls);
            } else {
                // 否则直接写入文件
                this.save_(urls);
            }
        }

        push_(urls) {
            urls.forEach((url) => {
                this.queue.push(url);
            });
        }

        save_(urls) {
            console.log('save\tstart ');

            this.isFlushing = true;

            fwrite({
                file: path.join(__dirname, 'urls.txt'),
                data: urls.join('\r\n') + '\r\n'
            })
            .then(() => {
                console.log('save\tend ');

                this.isFlushing = false;
                this.flush_();
            })
            .catch((err) => {
                this.error_(err);
            });
        }

        error_(err) {
            console.log(err.message);
        }

        flush_() {
            if (this.queue.length) {
                let urls = this.queue;

                this.queue = [];
                this.save_(urls);
            }
        }
    }
```

最后我们初始化我们的爬虫，即可以开始爬取数据了

```javascript
    let queue = [HOST_URL + '/browse/'];
    let record = new Record();
    let spider = new Spider(queue, record);
```