# 介绍
这是我修改自[hexo-theme-matery](https://github.com/blinkfox/hexo-theme-matery)的个性化hexo博客模板，主要修改了一些个性化配置，为了方便大家直接搭建使用。

# 我的博客演示
[https://godweiyang.com](https://godweiyang.com)

# 搭建教程
[https://godweiyang.com/2018/04/13/hexo-blog/](https://godweiyang.com/2018/04/13/hexo-blog/)

# 使用方法
为了减小源码的体积，我将插件目录`node_modules`进行了压缩，大家下载完后需要解压。另外添加水印需要的字体文件我也删除了，大家可以直接从电脑自带的字体库中拷贝。

* 首先运行`git clone git@github.com:godweiyang/hexo-matery-modified.git`将所有文件下载到本地。
* 解压`node_modules.zip`，然后删除`node_modules.zip`和`.git`文件夹。
* 还缺一个字体（为图片添加水印需要用到），去`C:\Windows\Fonts`下找到`STSong Regular`，复制到`hexo-matery-modified`文件夹下。

这样所有准备工作就做好啦，剩下的工作就直接去看我的教程[https://godweiyang.com/2018/04/13/hexo-blog/](https://godweiyang.com/2018/04/13/hexo-blog/)

# 使用问题

```
$ hexo g
INFO  Start processing
FATAL Something's wrong. Maybe you can find the solution here: https://hexo.io/docs/troubleshooting.html
TypeError: Cannot read property 'count' of null
    at Hexo.module.exports (D:\Dev\Blog\node_modules\hexo-baidu-url-submit\lib\generator.js:4:41)
    at Hexo.tryCatcher (D:\Dev\Blog\node_modules\bluebird\js\release\util.js:16:23)
    at Hexo.<anonymous> (D:\Dev\Blog\node_modules\bluebird\js\release\method.js:15:34)
    at Promise.map.key (D:\Dev\Blog\node_modules\hexo\lib\hexo\index.js:318:20)
    at tryCatcher (D:\Dev\Blog\node_modules\bluebird\js\release\util.js:16:23)
    at MappingPromiseArray._promiseFulfilled (D:\Dev\Blog\node_modules\bluebird\js\release\map.js:61:38)
    at MappingPromiseArray.PromiseArray._iterate (D:\Dev\Blog\node_modules\bluebird\js\release\promise_array.js:114:31)
    at MappingPromiseArray.init (D:\Dev\Blog\node_modules\bluebird\js\release\promise_array.js:78:10)
    at MappingPromiseArray._asyncInit (D:\Dev\Blog\node_modules\bluebird\js\release\map.js:30:10)
    at _drainQueueStep (D:\Dev\Blog\node_modules\bluebird\js\release\async.js:142:12)
    at _drainQueue (D:\Dev\Blog\node_modules\bluebird\js\release\async.js:131:9)
    at Async._drainQueues (D:\Dev\Blog\node_modules\bluebird\js\release\async.js:147:5)
    at Immediate.Async.drainQueues [as _onImmediate] (D:\Dev\Blog\node_modules\bluebird\js\release\async.js:17:14)
    at runCallback (timers.js:705:18)
    at tryOnImmediate (timers.js:676:5)
    at processImmediate (timers.js:658:5)

```

解决方式：

```

npm remove hexo-baidu-url-submit
hexo clean
hexo g

```

参考连接：https://blog.csdn.net/elgong/article/details/97839395