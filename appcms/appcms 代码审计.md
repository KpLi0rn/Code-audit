## Appcms 代码审计 

### 0x01 任意文件包含（前台RCE）

文件位置: `index.php` 大约202行左右

![image-20210113215845480](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113215845480.png)

发现在文件202行进行了路径的拼接，且`$tpl` 参数用户可控

![image-20210113215958368](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113215958368.png)

并没有对路径进行过滤，导致之间将文件包含到我们`/template/default/` 文件夹下

这里不过我们需要清楚一点，就是我们只是部分路径可控，比如我们上传了一个 `xxxxx.jpg` 那么在包含的过程中代码会拼接 `.php` 就会导致模版找不到的情况出现

![image-20210113224041607](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224041607.png)

所以如果想要包含我们的shell的话，我们需要利用php低版本的阶段来截断后面的 `.php` ，php版本小于 5.3.4 我这里的版本是`phpstudy的php 5.2.17`

接下来我们可以有两种办法

1. 包含中间件的日志（这套cms好像没有自己的日志）
2. 找前台可上传的点，上传我们的文件然后利用文件包含进行引入

我这里尝试了第二种，因为在实战过程中第一种的路径又需要信息搜集不是很稳，如果我们能有cms的前台上传点的话那么就很稳了

这里我翻了翻源码发现在 `upload` 文件夹下有两个文件

![image-20210113224502897](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224502897.png)

直接访问第一个文件会发现有权限校验

![image-20210113224539482](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224539482.png)

访问第二个文件发现是一个上传样式

![image-20210113224611260](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224611260.png)

图片上传之后发现没反应，f12查看源码，发现会将请求发送到 `/upload/upload_file.php?params=&v=dBJhuBrKulYUJ8wNxlliKrclH` 接口

![image-20210113224657660](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224657660.png)

这不直接本地梭哈一下

本地构造上传文件,点击上传，美滋滋文件上传成功了

```html
<form id="form1" enctype="multipart/form-data" method="post" action="http://192.168.1.13//upload/upload_file.php?params=&v=3udxH=Y=3WK9P0tRR2pLEBBVe">
    <input type="file" name="Filedata">
    <input type="submit" value="Upload">
</form>
```

同时在url中返回了重新命名的文件名

![image-20210113224827056](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224827056.png)

url解码一下

![image-20210113224839149](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224839149.png)

然后回到我们的index文件，直接包含+截断

![image-20210113224902297](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224902297.png)

美滋滋啊 真不戳

![image-20210113224930727](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113224930727.png)

嘿嘿既然发现了接口未授权我们就去看一下好了 

文件 `/upload/upload_file.php`

发现这里源码是根据get和v参数接受到的参数进行鉴权的，看来是大意了呀，还是得从session中进行鉴权

![image-20210113230705171](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113230705171.png)

那么再去看 `upload_form.php` 为啥会显示get和v参数的值

发现这里35行直接获取了，其实主要是这个文件没有做好权限的校验，导致我们能获取get和v从而上传文件

![image-20210113230857303](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210113230857303.png)

### 0x02 部分SSRF+文件读取

文件位置:`/pic.php` 

读取非php文件源码，利用SSRF探测内网开放情况，访问默认内网页面

这尼玛就离谱，搞了半天

可以看到这里作者还是做了一些过滤的

首先会对我们传入的参数进行一个base64的解码

![image-20210114220541433](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210114220541433.png)

然后代码中会先用 `.` 对传入的url进行一个拆分，然后会取最后一个 `.`后的判断是否在数组中

这种情况我们还是比较好绕过的可以利用如下 

`?q=1.jpg` 

接下来判断里面有没有php，如果有那么就不行

这里真的绕不过去，看网上有师傅是利用url编码绕过的，但是我本地不行，后面重新测试了一下感觉确实是不行 

在我们浏览器发送请求的时候 服务器会对我们URL中的参数进行一次url解码，但是这里传输的是base64 

后端只会接收到base64然后进行解码，这中间是不会再进行一次url解码的

不过根据目前的情况还是有一些姿势能读取部分文件，ssrf，和host头注入

1. 部分文件读取

这里读取的是`index.php` 因为访问目录的时候默认访问的是 index文件，所以我们可以构造 

`http://localhost/?q=1.jpg` 进行读取主页的源码，或配置文件等

![image-20210115180021700](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115180021700.png)

2. 部分ssrf

同样由于php的限制我们只能访问 index这样的页面 

`http://localhost/?q=1.jpg`



发现在第17行的时候，host头可控，正常情况下可以利用%0a%0d 进行CRLF攻击

![image-20210115143113953](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115143113953.png)

但是`header` 函数自4.4之后就增加了对host头攻击的保护 ，所以这里是不会存在CRLF注入等，这里感谢丁师傅的提示

![image-20210115143250308](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115143250308.png)

### 0x03 SQL注入 ？

文件位置 `comment.php`

个人感觉不存在，理由如下

首先 `$page['post'] ` 接受的是post接受到的数据，按照如下格式存储在`page['post'] ` 中

```php
'ip'=>'xxxx','host'=>'localhost'
```

![image-20210115203340451](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115203340451.png)

继续往下看 ，发现对传入对参数都进行了html实体化处理

这样我们无法通过闭合前面的特殊符号来使得我们的sql语句逃逸出来从而执行，继续往下看，因为此时我们还不知道sql语句的结构

![image-20210115203710894](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115203710894.png)

发现在80行我们的评论进入了`filter_words` 函数 

跟进一下发现只是对敏感词的一个过滤 

![image-20210115204014646](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115204014646.png)

继续往下看发现调用了 `single_insert` 来执行我们的语句 

![image-20210115204104870](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115204104870.png)

我们跟进去看一下sql语句的结构 

发现我们传入的 `$fields` 会进行一个循环处理，然后对我们的数值进行一个 `' '` 的添加，key是不会添加 `'`但是key我们并没有办法控制，我们只能控制 `value` ，在html实体转义之后我们输入的引号不会闭合之前的结构，所以我个人觉得这里是不存在SQL注入的，同样的ip也没做过滤但是我们仍然无法跳脱出引号。有可能我这个版本是漏洞修复后的吧

![image-20210115204200001](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115204200001.png)

### 0x04 反射XSS 

文件位置 : `callback.php` 

没有进行过滤2333 拿下！

![image-20210115212959547](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115212959547.png)

简单构造一下即可 

![image-20210115213029775](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115213029775.png)

![image-20210115213043490](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210115213043490.png)

### 0x05 模版注入（后台RCE）

文件位置：`/admin/template.php`

`$page['post']['content']` 可控，所以我们可以在编辑过程中插入我们的恶意代码

![image-20210117095932192](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210117095932192.png)

![image-20210117102316003](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210117102316003.png)

