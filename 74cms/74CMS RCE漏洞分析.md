## 74CMS RCE漏洞分析

### 0x00 前言

之前在逛p牛知识星球的时候看到了panda师傅发的这个74cms文件包含RCE，然后又是最近的所以就跟进一波，从中学习学习

复现环境以及源码下载链接：

win10 、phpstudy

源码下载链接：http://data.wjlshare.xyz/project-source-code/74cms_Home_Setup_v6.0.4.zip

### 0x01 前置知识

74cms是基于ThinkPHP 3.2.3框架的基础上二次开发的，所以我们在审计74cms之前我们需要先对ThinkPHP框架做一个熟悉，ThinkPHP的开发手册写的非常好，我放到我的服务器上了下载链接如下

下载链接：http://data.wjlshare.xyz/project-source-code/ThinkPHP3.2.3dev.pdf    

ThinkPHP是一个单入口文件进行项目的部署和访问，所有功能的实现都是通过这个唯一的入口文件。

![image-20210109180758031](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109180758031.png)

大致项目结构如下：

![image-20210109181339812](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109181339812.png)

ThinkPHP中主要采用的是多层MVC架构：Views(视图)、Model(模块)、Controller(控制器)



Model: 主要负责封装和实现应用的具体功能等

Views：主要负责视图的展现

Controller：主要负责请求拦截妆发，加载配置文件，调配对应的模块来处理对应的请求等

ThinkPHP框架自带了一个应用目录结构，并且带了一个默认的应用入口文件，方便部署和测试，默认的应
用目录是Application

![image-20210109182639678](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109182639678.png)

ThinkPHP项目在初始化的时候会自动的生成几个默认的模块

![image-20210109183238527](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109183238527.png)

Common 公共模块是不能直接访问的 

其中每个模块之间都是独立的，所以每个模块都可以自由的设置和添加 (也就是之前说的多层MVC架构)

![image-20210109185006152](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109185006152.png)

ThinkPHP下访问是通过利用入口文件访问对应模块下对应控制器的对应操作方法

ThinkPHP下访问默认页面就会通过如下路由进行访问

`http://localhost/index.php?m=Home&c=Index&a=index`

m : Home模块

c ：Index控制器类 

a ：index操作

![image-20210109185735953](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109185735953.png)

效果如下：

![image-20210109190440864](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109190440864.png)

我们这里直接来看74cms这样更加形象一些



![image-20210109190633733](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109190633733.png)

上面这个路由对应访问的就是

Admin模块下的Index控制类中的login操作

![image-20210109190807254](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109190807254.png)

这边只能简单的介绍一下ThinkPHP的一些信息，具体的话师傅们可以去看开发者手册

### 0x02 漏洞复现

#### 文件包含日志实现RCE

利用报错将shell写入日志

![image-20210108163807087](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210108163807087.png)

由于基于thinkPHP 二次开发，所以日志目录就是 ThinkPHP的日志目录

`/data/Runtime/Logs/Home/xx_xx_xx.log`

![image-20210108163828660](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210108163828660.png)

这里如果直接是 `<?php phpinfo();?>` 是不会有内容回显的，为什么呢？我们在下面的代码分析中会提到

### 0x03 代码分析

我们来看74cms官网修复的地方

![image-20210109210212287](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109210212287.png)

漏洞代码如下：

![image-20210109210303204](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109210303204.png)

可以发现官方修复手法在之前的基础上增加了一个判断 



我们从漏洞点开始分析：

文件位置：`/Application/Common/Controller/BaseController.class.php` 大约 168行左右

发现`assign_resume_tpl` 函数需要传入两个参数，文件包含存在的位置在 `$tpl` 参数，所以我们直接跟进 fetch方法 

![image-20210109210425387](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109210425387.png)

文件位置 ： `/ThinkPHP/Library/Think/Controller.class.php`  , 之前传入的 `$tpl` 传到了这里的第一个参数的位置，`$content`  `$prefix` 都为空，继续跟进

![image-20210109210945718](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109210945718.png)

文件位置： `/ThinkPHP/Library/Think/View.class.php` 

由于`$content` 参数为空，所以进入if判断语句，如果文件存在就进入 `parseTemplate` ,此时的 `$templateFile` 的数值为我们之前传入的文件地址 例 ：` ./data/upload/resume_img/2101/08/5ff814074d7af.png`，由于文件调用 `parseTemplate` 进行解析

```php
$templateFile   =   $this->parseTemplate($templateFile);
```

![image-20210109211312929](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109211312929.png)

跟进到我们的 `parseTemplate`  方法，在同一个文件

这个函数会进行一个判断传入目标路径是否是文件，如果是的话则直接进行一个返回，如果不是文件的话就会利用ThinkPHP的C函数读取配置文件中的`TMPL_TEMPLATE_SUFFIX` 进行一个拼接

默认拼接后缀为 `.html` 

![image-20210109211909708](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109211909708.png)

这里我们传入的文件路径是文件所以直接返回路径，并不会进行拼接

![image-20210109211724567](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109211724567.png)



继续返回到之前的fetch函数

继续往下看发现有一个if判断语句会将`php`和 `TMPL_ENGINE_TYPE` 进行一个比较 ，通过查看发现74cms默认的值为THINK

![image-20210109212510764](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109212510764.png)

所以我们进入到else语句中，将数值传入了数组，然后再带入到了 `Hook::listen` 方法中

即监听 `view_parse`  标签位，利用数组传入多个参数

![image-20210109212358136](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109212358136.png)

ps: 这里来解释一下上面的那个问题 为什么 普通的 `<?php phpinfo();?>` 没有回显，而是需要在后面添加 `ob_flush();`

我们这里看到第118行，`ob_start();` 这个代码的意思是打开缓冲区，所以我们phpinfo信息会存储在缓冲区，所以才没有回显，那么我们想要有回显的话，只需要执行ob_flush冲刷出（送出）输出缓冲区中的内容，打印到浏览器页面上即可

参考链接：https://xz.aliyun.com/t/8596#toc-5

这里的标签涉及ThinkPHP框架的Behavior行为，我们可以把标签理解为钩子函数，即程序运行到这个标签的时候就会被拦截下来执行相关的行为 

![image-20210109215920344](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109215920344.png)

继续跟进文件

文件位置：`/ThinkPHP/Library/Think/Hook.class.php` 大约80行左右的位置

当系统执行到了 `view_parse` 标签位的时候，ThinkPHP会找到`Hook::listen()` 方法，查找 `$tags` 中绑定 `view_parse` 标签的方法，然后利用foreach进行遍历并且执行 `Hook:exec` 方法

![image-20210109220103078](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109220103078.png)

跟进 `exec` 方法 约119行左右

发现会做一个判断检查行为名称，如果包含 `Behavior` 字符串，那么入口就必须是run方法

![image-20210109221447689](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109221447689.png)

全局搜索 `view_parse`  找到配置文件

文件位置：`/ThinkPHP/Mode/common.php` 大约61行左右

![image-20210109221714998](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109221714998.png)

从上面的配置文件中可以发现，`view_parse`   执行了 `Behavior\ParseTemplateBehavior` 类，所以入口就是`run` 方法，我们跟踪过去进行查看

文件位置 : `/ThinkPHP/Library/Behavior/ParseTemplateBehavior.class.php` 大约17行左右

上文我们已经知道我们的模板引擎就是Think所以我们进入第一个if 判断，如果我们是第一次进行模板解析的话会进入else语句，然后执行了`fetch()` 方法

![image-20210109222218884](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109222218884.png)

继续跟踪

文件位置：`/ThinkPHP/Library/Think/Template.class.php` 大约75行左右

发现在77行左右调用了 `loadTemplate` 方法

![image-20210109222601472](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109222601472.png)

继续跟进我们的方法在同文件下大约89行左右

会进行一个if判断如果我们是文件的话就会读取我们的内容 存放到 `$tmplContent` 中，由于默认 `LAYOUT_ON` 是false 所以我们直接进入else语句

![image-20210109222750913](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109222750913.png)

继续跟踪我们的`compiler` 函数

同名文件下 大约129行左右

发现在解析过程中，并没有对文件内容进行过滤，而是直接进行了拼接，然后返回了我们的内容

![image-20210109223013875](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223013875.png)

回到我们之前的 `loadTemplate`函数

将我们之前返回的内容进行一个缓存处理

![image-20210109223413736](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223413736.png)

全局搜索 `cache_suffix`

![image-20210109223551025](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223551025.png)

![image-20210109223513777](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223513777.png)

即如下位置

![image-20210109223629502](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223629502.png)

最后返回我们的缓存文件名给fetch函数

![image-20210109223804266](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223804266.png)

如下图：

![image-20210109223844377](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223844377.png)

进入了 `Storage::load($templateCacheFile,$this->tVar,null,'tpl');` 方法

我们跟进该方法

文件位置：`/ThinkPHP/Library/Think/Storage/Driver/File.class.php` 大约76行左右

发现包含了这个文件，从而导致RCE

![image-20210109223954100](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210109223954100.png) 







### 0x04 参考链接

https://xz.aliyun.com/t/8596#toc-5

https://xz.aliyun.com/t/8520











