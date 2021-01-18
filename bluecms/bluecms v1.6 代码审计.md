## BlueCMS V1.6 代码审计

### 0x00 前言

之前因为种种原因一直没有开始php的审计（其实就是因为自己懒），现在趁着寒假的时间来搞一波，这个cms是之前先知上看到了，也看到很多人都用这个cms开头所以也来审计一下

### 0x01 思路

由于本人也是审计小白所以有的地方有误还请多多提出。

由于cms的文件、文件夹都会很多，所以我会事先简单的看一下各个文件夹下的php大致功能，然后我再依次文件看下来，不过一般都是从入口文件开始查看

项目结构如下：

![image-20210105135817608](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105135817608.png)

这里主要的代码都在uploads下所以我们直接看uploads下即可

这里参考：https://xz.aliyun.com/t/7992?spm=5176.12901015.0.i12901015.5277525cY8DiJc

```
├── admin     后台管理目录
├── install   网站的安装目录
├── api       接口文件目录
├── data     系统处理数据相关目录
├── include  用来包含的全局文件
└── template  模板
```

### 0x02 漏洞挖掘

在安装过程中查看install文件夹发现配置文件 data/confg.php

![image-20210105141342155](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105141342155.png)

在配置文件中发现项目采用的编码格式为 GBK2312 ，看到这个编码后面在看SQL语句过程中就可以多留心一下了，因为可以通过 %df 来进行宽字节注入来绕过php的转义

![image-20210105141436156](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105141436156.png)

在对框架以及一些配置文件熟悉了之后就可以开始正式审计了，我这里直接从第一个文件开始看

### 0x03 ad_js.php

![image-20210105141805829](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105141805829.png)

在第二行我们发现该文件引入了/include/common.inc.php文件，我们追踪过去看看。

在开头部分我们可以看到这里定义了三个参数，定义了文件的根目录，上传目录以及数据处理相关目录

![image-20210105142024209](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105142024209.png)

下面发现引入了一些功能

![image-20210105142148089](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105142148089.png)

这个结构其实就是对所有传入的参数都进行了转义处理，会对传入的 " 和 ' 进行转义处理

![image-20210105142225661](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105142225661.png)

所以引入这个文件其实就是引入了一些基本设置，过滤规则等功能

#### UNION SQL注入漏洞 

文件位置 `/uploads/ad_js.php`

 $_GET获取到的数值并没有进行处理，直接拼接到了后面的SQL语句并进行了执行

![image-20210105143212554](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105143212554.png)



利用如下：

回显点为7

![image-20210105145239089](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105145239089.png)

`/php/bluecms/uploads/ad_js.php?ad_id=-1+union+all+select+1,2,3,4,5,6,database()`

![image-20210105145257567](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105145257567.png)



#### 反射型XSS漏洞

文件位置 `/uploads/ad_js.php`

由于引入的common.inc.php 文件中，为显示ERROR报错信息，从而导致直接将我们输入的信息进行了输出

```php
error_reporting(E_ERROR);
```

![image-20210105145816591](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105145816591.png)

```php
$ad_id = !empty($_GET['ad_id']) ? trim($_GET['ad_id']) : '';
```

![image-20210105150110505](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105150110505.png)

### 0x04 comment.php

#### INSERT SQL注入漏洞

问题出在getip() 函数上

![image-20210105151004488](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105151004488.png)

发现获取ip过程中没有对数值进行过滤

![image-20210105151102641](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105151102641.png)

所以只要是带有getip() 函数的地方就会存在SQL注入

根据代码进行构造

![image-20210105151351166](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105151351166.png)

构造如下payload

`888' and if(ascii(substr(database(),1,1)=98),sleep(5),1),'1')#`

![image-20210105151907298](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105151907298.png)

### 0x05 guest_book.php

#### INSERT SQL注入漏洞

同样的这里问题出在onlineip上

![image-20210105152651154](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105152651154.png)

$online_ip 其实就是getup() 所以实际上出问题的还是getip()

![image-20210105152708558](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105152708558.png)

还是同样的配方hhhh

`888' and if(ascii(substr(database(),1,1)=98),sleep(5),1),'1')#`

![image-20210105153019665](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105153019665.png)



### 0x06 publish.php

#### INSERT SQL注入漏洞

原理还是因为getip()没有做好过滤导致

![image-20210105171622921](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105171622921.png)

payload如下：

`888' and if(ascii(substr(database(),1,1)=98),sleep(5),1) or '1'='1`

![image-20210105170706246](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105170706246.png)

#### 反射型XSS漏洞

原因和之前一样，因为将错误回显了出来，所以导致可解析payload从而导致xss （不过这里位置是X-Forward-For 所以漏洞危害非常小）

注意要加入引号或其他的使其报错

![image-20210105164705643](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105164705643.png)



#### 任意文件删除漏洞

由于id参数没有进行路径限定，同时又直接和项目路径进行了拼接，从而导致任意文件删除

![image-20210105171800678](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105171800678.png)

只需构造如下格式即可删除文件 

`http://192.168.1.9:8888/php/bluecms/uploads/publish.php?act=del_pic&id=q.txt`

执行之后q.txt文件便会被删除

### 0x07 user.php

#### 任意URL跳转（比较鸡肋）

第16行，我们传入的参数可控

![image-20210105213312129](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105213312129.png)

然后再进行一次base64解码

![image-20210105213413497](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105213413497.png)

最后来到我们的漏洞点，直接传入了我们可以控制的$from

![image-20210105213432647](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105213432647.png)

burp抓包将请求包中的from进行修改 

![image-20210105213732294](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105213732294.png)

释放之后则会跳转到百度

![image-20210105213947612](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105213947612.png)

#### 存储XSS漏洞

我们来看到第268行，发现将我们的数值传入到了filter_data中

![image-20210105175909698](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105175909698.png)

跟踪 filter_data函数，我们可以发现这里过滤没有做好，我们有很多中方式可以绕过～

![image-20210105180001958](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105180001958.png)

直接在文章概要中插入我们的xss payload即可

![image-20210105181144207](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105181144207.png)

可以发现存储xss已经写进去了

![image-20210105181258942](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105181258942.png)

#### 任意文件删除漏洞

同样是因为传入参数可控导致，这里直接通过post接受lit_pic参数的数值并且进行了一个拼接，如果文件存在则删除

![image-20210105182000228](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105182000228.png)

构造如下payload即可

post内容 `post_id=1&link_man=aaa&link_phone=13888&link_email=123@123&link_qq=1234&link_address=cccc&lit_pic=./1.txt&title=aaa`

![image-20210105194602932](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105194602932.png)



#### 任意文件包含漏洞

我们来看到这个代码，发现include中部分路径是我们可以控制的，那么在特定版本下我们可以实现任意文件包含 

在php 5.4版本下我们可以截断后面的 /index.php 来实现任意文件包含

![image-20210105194811739](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210105194811739.png)

只需要如下即可进行截断

```
pay=..%2F..%2Frobots.txt....................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
```

可以通过上传图片马进行包含从而getshell

#### 宽字节 SQL注入

该文件在头部引入了 common.inc.php 文件

```php
require_once dirname(__FILE__) . '/include/common.inc.php';
```

前面分析过 common.inc.php 引入会对传入的参数都进行一个转义但是由于编码是gbk所以我们可以通过宽字节注入来实现 %df

在代码的第862行开始利用$_GET获取前端的user_name参数的数值

![image-20210106114131426](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106114131426.png)

来看具体功能点，根据源码分析，在判断用户名是否存在的时候会进行两次SQL查询，会通过这个接口进行判断

![image-20210106114328779](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106114328779.png)

![image-20210106190006183](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106190006183.png)

![image-20210106094717188](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106094717188.png)

当我们输入%df' 的时候会发现报错 

利用宽字节来闭合前面的 \' 

![image-20210106114441577](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106114441577.png)

payload如下 发现了时延

`1%df' and (select 1 from(select case when(ascii(substr(database(),1,1))=98) then sleep(5) else (1) end)x) -- -`

因为这里在代码中是会执行两次sql查询语句的所以这里sleep的时间是双倍的

![image-20210106114740286](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106114740286.png)



#### 存储型XSS漏洞

136行处起的代码对传入的参数都没有进行严格的过滤

![image-20210106173022943](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106173022943.png)

这里选取邮箱来举例，可插入类似 `123@123.<svg/onload=alert(1)>` 的payload

只需在注册的时候将邮箱改成上述payload即可

![image-20210106173255432](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106173255432.png)

#### INSERT SQL注入

还是一样的问题由于没有对email那块的参数进行过滤导致，可以通过构造进行数据的插入

![image-20210106192931224](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106192931224.png)

利用burp进行抓包，修改email ，通过测试发现如果有 123@123.x前端会导致注入失败，所以利用burp绕过前端的邮箱验证

payload如下 ，进行多行的插入

`%df',1,1),(88,1223,md5(123456),(select database()),1,1) #`

![image-20210106193152356](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106193152356.png)

利用账户名 1223 密码 123456 进行登陆，发现database已经注入出来了

![image-20210106193244956](https://gitee.com/KpLi0rn/BlogImag/raw/master/img/image-20210106193244956.png)





### 0x08 总结

这个毕竟是十年前的cms了 漏洞自然非常多，第一次审cms感觉收获还是比较多的，以前总看着代码很多就退缩了，但是其实耐心看还是能捋清楚的。

ps：其中还存在很多这样的漏洞，其实就是开发者以为转义就万事解决了，但是由于编码设置失败的问题导致可以利用宽字节进行注入，其他还有很多问题，比如注册可使用同一个邮箱导致覆盖万能密码这种等等