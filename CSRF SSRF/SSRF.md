https://xz.aliyun.com/t/12227

### 0x00 定义与成因

**SSRF**(Server-Side Request Forgery:服务器端请求伪造) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞。一般情况下，SSRF攻击的目标是从外网无法访问的内部系统。（正是因为它是由服务端发起的，所以它能够请求到与它相连而与外网隔离的内部系统）

SSRF 形成的**原因**大都是由于服务端提供了从其他服务器应用获取数据的功能且没有对目标地址做过滤与限制。比如从指定URL地址获取网页文本内容，加载指定地址的图片，下载等等。

**目的**

- 可以对外网、服务器所在内网、本地进行端口扫描，获取一些服务的banner信息
- 攻击运行在内网或本地的应用程序(比如溢出)
- 对内网WEB应用进行指纹识别，通过访问默认文件实现
- 攻击内外网的web应用，主要是使用GET参数就可以实现的攻击(比如Struts2,sqli等)
- 利用file协议读取本地文件等

**漏洞点**

能够对外发起网络请求的地方，就可能存在SSRF漏洞
从远程服务器请求资源(Upload from URL，Import & Export RSS Feed)
数据库内置功能(Oracle、 MongoDB、MSSQL、 Postgres、 CouchDB)
Webmail收取其他邮箱邮件(POP3、 IMAP、SMTP)
文件处理、编码处理、属性信息处理(ffmpeg、 ImageMagic、 DOCX、 PDF、 XML)

**常见场景（具体漏洞点）**

1、通过URL地址进行网页分享;

```
http://share.xxx.com/index.php?url=http://www.xxx.com
```

2、转码服务，通过URL地址把原地址的网页转换格式

3、图片加载与下载，一般是通过url参数进行图片获取

```
http://image.xxx.com/image.php?image=http://www.xxx.com
```

4、未公开的api实现以及其他调用url的功能;

5、设备后台管理进行存活测试;

6、远程资源调用功能;

7、数据库内置功能;

8、编辑器进行远程图片抓取，如: ueditor;

9、打包附件或者内容编辑并导出时

10、PDF生成或导出

**危害**

1、可以对服务器所在的内网环境进行端口扫描、资源访问 

2、利用漏洞和Payload进一步攻击运行其他的应用程序

3、对内网web应用进行指纹识别，通过访问应用存在的默认文件实现 

4、GET型漏洞利用，GET参数就可以实现的攻击，比如struts2漏洞利用等 

5、POST型漏洞利用，可利用gopher协议进行参数构造

6、利用Redis未授权访问getshell、Weblogic默认SSRF漏洞页面 

7、如果ssrf漏洞存在于云服务器

### 0x01 漏洞函数

```php
file_get_contents()
//这个函数的作用是整个文件读入一个字符串中

//example
<?php
echo file_get_contents(“test.txt”);
?>
//result:输出test.txt文件里面的字符串。
```

```php
fsockopen(
    string $hostname,
    int $port = -1,              //端口号，省略/-1表示不使用端口
    int &$error_code = null,  
    string &$error_message = null,  //错误信息
    ?float $timeout = null   //连接时限，单位s
): resource|false
// 打开 Internet 或者 Unix 套接字连接，返回文件指针
    
//exapmle通过UDP服务（端口号 13）中来检索日期和时间。
<?php
$fp = fsockopen("udp://127.0.0.1", 13, $errno, $errstr);
if (!$fp) {
    echo "ERROR: $errno - $errstr<br />\n";
} else {
    fwrite($fp, "\n");
    echo fread($fp, 26);
    fclose($fp);
}
?>
```

```php
curl_exec($handle): string|bool
//执行指定 cURL 会话。
//参数handle：由 curl_init() 返回的 cURL 句柄。
    
//example获取网页
<?php
// 创建新的 cURL 资源
$ch = curl_init();

// 设置 URL 和相应的选项
curl_setopt($ch, CURLOPT_URL, "http://www.example.com/");
curl_setopt($ch, CURLOPT_HEADER, 0);

// 抓取 URL 并把它传递给浏览器
curl_exec($ch);

// 关闭 cURL 资源，并且释放系统资源
curl_close($ch);
?>
```

### 0x02利用协议

```php
Http协议   //直接访问http资源
http://xxx.com/api/readFiles?url=http.//10.1.1.1/x

File协议   //进行服务器文件读取
http://xxx.com/api/readFiles?url=file:///ete/passwd

Dict协议    //可用此协议进行端口开放探测
http://xxx.com/api/readFiles?url=dict//1000.1:22

Gopher协议  //gopher支持发出GET、POST请求，可进行复杂的漏洞利用
//example 漏洞URL：http://xxx.com/get.php?name=admin
//gopher协议进行请求gopher://192.168.1.120:80/ GET%20/get.php%3fname=admin%20HTTP/1.1%0d%0aHost:xxx.com%0d%0aRequest
{
    //上面URL对应的请求体
    GET /get.php?name=admin HTTP/1.1Host:192.168.1.120
}
```

### 0x03常用绕过方式

1.要求域名http://www.xxx.com不可更改

使用@绕过

```
http://www.xxx.com@10.10.10.10，则实际上访问的是 10.10.10.10
```

2.要求固定后缀比如.jpg

使用#绕过

```
http://10.10.10.10:5001/#/abc.jpg
实际在浏览器访问的是 http://10.10.10.10:5001
```

3.限制请求的IP不为内网地址

**点分割符号替换**

```
可以使用。 代替域名分割中的.
比如127.0.0.1
```

**xip.io**

```
10.10.10.10.xip.io 会被解析成10.10.10.10
```

**十进制ip地址**

```
127.0.0.1的十进制: 2130706433，HTTP访问: http://2130706433/
//IP地址是个32位的二进制数，表示成点分10进制
```

**进制转换**

```
127.0.0.1的八进制: 0177.0.0.1，十六进制: 0x7f.0.0.1
```

**封闲式字母数字**

```
 ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ >>> example.com
​ ①②⑦. ⓪.⓪.①>>> 127.0.0.1
```

**302重定向**

```php
//需要vps，把302转换的代码部署到vps上，然后访问

//vps端代码如下
<?php 
header("Location: http://192.168.1.10");
exit(); 
?>
```

**绕过localhost**

```
http://[::1]
```

**DNS重绑定 短网址**https://xzfile.aliyuncs.com/upload/affix/20230227000356-2c959fd2-b5ef-1.pdf

### 0x04漏洞案例

**1.DVWA**

**2.见参考链接**

https://xzfile.aliyuncs.com/upload/affix/20230227000356-2c959fd2-b5ef-1.pdf

### 0x05加固和防御

1.去除url中的特殊字符
2.将域名解析为IP，对内网IP进行限制
3.不跟随30x跳转（跟随跳转需要从1开始重新检测）
4.禁用高危协议，例如：gopher、dict、ftp、file等，只允许http/https
5.请求时设置host header为ip
6.统一错误信息，避免用户可以根据错误信息来判断远程服务器的端口状态。