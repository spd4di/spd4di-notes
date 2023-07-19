### CSRF-Cross Site Request Forgery

**危害**

以用户的名义发邮件/发消息；

转换/购买商品；

修改密码;

删除文章等

**原因**

- http协议使用session在服务端保存用户的个人信息,客户端浏览器用cookie标识用户身份;
- 用户登录了某个web站点,同时点击了包含CSRF恶意代码的URL,就会触发CSRF

**条件**

- 用户登录A网站，生成cookie
- 不登出的同时访问了恶意URL

**防御**

- 添加验证码
- http头使用token/referer
- 修改密码需要原密码
- 验证码

#### 1.get类型CSRF

```
//银行站点A使用GET请求转账
http://www.mybank.com/Transfer.php?toBankId=11&money=1000 

//攻击者使用站点B，包含以下HTML代码
<img src=http://www.mybank.com/Transfer.php?toBankId=11&money=1000>
```

#### 2.post型CSRF

#### CSRF漏洞挖掘

1.没有referer/token

2.删除referer提交仍然有效

3.CSRF Tester

