### XML--可拓展标记语言*EX*tensible *M*arkup *L*anguage

XML被设计用来传输和存储数据，而非显示数据

difference between XML/HTML

- XML 被设计为传输和存储数据，其焦点是数据的内容。

- HTML 被设计用来显示数据，其焦点是数据的外观。

- HTML 旨在显示信息，而 XML 旨在传输信息。

XML的特点和语法规则

- 自我描述性

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
// XML 声明
<note>
//描述文档的根元素
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
//根的 4 个子元素
</note>
//根元素的结尾
```

- XML文档形成一种树结构

- 所有XML元素都须有关闭标签
- XML标签对大小写敏感

- XML属性须加引号
- XML会保留空格，HTML 会把多个连续的空格字符裁减（合并）为一个



XML注释

```
<!-- This is a comment --> 
和HTML注释类似
```

```
//内部实体
//<!ENTITY 实体名称 "实体的值">

<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe "test" >]>         //内部实体声明
<creds>
<user>&xxe;</user>       //内部实体引用
<pass>mypass</pass> 
</creds>
```

```
//外部实体
//<!ENTITY 实体名称 SYSTEM "URI/URL">

<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///c:/test.dtd" >]>      ///外部实体声明
<creds>
    <user>&xxe;</user>              //外部实体引用
    <pass>mypass</pass>
</creds>

```

DTD

```
//内部DTD
// <!DOCTYPE 根元素 [元素声明]>

<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [                    //DTD BEGIN
<!ELEMENT foo ANY >
<!ENTITY xxe "test" >]>            //DTD END
<creds>
<user>&xxe;</user>       
<pass>mypass</pass> 
</creds>
```

```
//外部DTD
//<!DOCTYPE 根元素 SYSTEM "文件名">

<?xml version="1.0"?>
<!DOCTYPE note SYSTEM "note.dtd">      //外部DTD
<note>
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note> 
```

