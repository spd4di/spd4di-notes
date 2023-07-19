### 介绍

jwt(JSON Web Token)是一串json格式的字符串，由服务端用加密算法对信息签名来保证其完整性和不可伪造。Token里可以包含所有必要信息，这样服务端就无需保存任何关于用户或会话的信息，JWT可用于身份认证、会话状态维持、信息交换等。

#### 组成

一个jwt token由三部分组成，header、payload与signature，以点隔开

:bear:JWT支持将算法设定为“None”。如果“alg”字段设为“ None”，那么签名会被置空,这样生成的任何token都是有效的（没有signature认证防止数据篡改）

:smile:可以将alg字段设置为None来伪造任何token，之后就可以使用伪造的token冒充访问任意用户登录网站

```js
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
#header经过base64URL编码，用于声明token的类型和前面算法
#eyJhbGciOiJOb25lIiwidHlwIjoiand0In0解码如下
{   
	"alg": "HS256",   #签名的算法algorithm，默认HS256
    "typ": "JWT"      #type令牌的类型，JWT令牌统一为JWT
}
#payload经过base64URL编码，用于表示真正的token信息
#eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ解码为
 {   
 	"sub": "1234567890",  
	"name": "John Doe",
	"iat": 1516239022 
}
JWT 规定了7个官方字段，供选用
iss (issuer)：签发人
exp (expiration time)：过期时间
sub (subject)：主题
aud (audience)：受众
nbf (Not Before)：生效时间
iat (Issued At)：签发时间
jti (JWT ID)：编号
#signature
前两部分用alg指定的算法加密，再base64URL加密就是signature了，作用是防止数据篡改
```

#### 解码

http://jwt.io/

#### JWT的安全问题

1.修改算法为none
2.修改算法从RS256到HS256
3.信息泄漏 密钥泄漏
4.爆破密钥