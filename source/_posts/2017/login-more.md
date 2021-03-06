---
title: 关于登陆的那些事
date: 2017-03-15 10:17:50
updated: 2017-03-15 10:17:50
tags:
categories:
---
### 普通的登录

这个是极其普通的登录需求，要的就是一个登录页面，输入账号密码，提交Form表单，后端查询数据库对应用户名的密码，匹配正确则把用户记录到Session，不正确则返回错误。
这种登录，在上学的时候，也许敬爱的老师就已经教过你了。
但可能他没有教你的是，密码需要hash加密，session为什么可以记录登录用户的原理。

#### 密码Hash
密码hash，就是存进数据库的密码是一串密文，密文是明文密码通过不可逆算法得出的。在Nodejs中，你可以使用bcryptjs，它提供了hash以及对应的compare方法，非常适合用于密码的加密和对比。

#### Session原理
Session的原理其实还是依赖了Cookie，所以Cookie才是记录用户凭证的真理。它的原理大概是酱紫的：服务器端维护一个session的表，这个表的每一条记录存的就是与某一个客户端的会话，会话会有过期时间，过期的会话会被清理。然后这个会话，会有一个对应的id，一般是一串长长的看不懂的字符串，然后这个字符串会被存储在客户端的cookie中，每一次请求服务器端都会带上这个cookie，服务器端就知道访问的就是哪个客户端了。
欲知更多有关「Session原理」请点击传送门：Session原理

### 使用独立登录系统

应项目需要，登录逻辑需要独立出来做成一个系统，就是另外一个项目。与原来的主站不是在同一个项目中了。一个域名是 www.site.com，一个则是passport.site.com了。要在不同的域名下进行登录，一般的方法是www.site.com/login 跳转到 passport.site.com/login，passport这边是一个登录页面，用户输入账号密码登录成功之后，passport会通过带着一个可逆加密的包含用户信息的token，重定向到www.site.com提供的回调处理地址，然后进行解密，匹配正确，则登录用户。
要注意的是，这里的加密的信息需要包含一个时间戳，接收方需要认证这个时间戳，过期登录失败。避免token被窃取，被无限登录site系统。

### 单点登录

单点登录需要实现的需求，说白了就是在站点A的登录了，那么用户就自动在站点B、站点C、站点E、F、G登录。
这又分两种情况，A站点和B站点是否在同一个二级域名下。
假如是在同一个域名下，例如`siteA.site.com`与`siteB.site.com`，因为cookie允许设置到二级域名下`.site.com`，所以siteA和siteB是可以共享cookie的，用户的信息可以通过可逆加密放在二级域名下的cookie，并且设置http only，就可以一站登录，站站登录。
而如果A站点和B站点不在同一二级域名下，例如`www.siteA.com`与`www.siteB.com`，他们就无法通过共享cookie的方式共享用户信息，所以需要用到jsonp的方式，用户在siteA登录之后，提供一个jsonp接口获取加密的用户信息，siteB访问这个jsonp获取加密信息。达到共享用户状态的效果。

单点登录SSO（Single Sign On）说得简单点就是在一个多系统共存的环境下，用户在一处登录后，就不用在其他系统中登录，也就是用户的一次登录能得到其他所有系统的信任。单点登录在大型网站里使用得非常频繁，例如像阿里巴巴这样的网站，在网站的背后是成百上千的子系统，用户一次操作或交易可能涉及到几十个子系统的协作，如果每个子系统都需要用户认证，不仅用户会疯掉，各子系统也会为这种重复认证授权的逻辑搞疯掉。实现单点登录说到底就是要解决如何产生和存储那个信任，再就是其他系统如何验证这个信任的有效性，因此要点也就以下两个：

>- 存储信任
>- 验证信任

#### 以Cookie作为凭证媒介

最简单的单点登录实现方式，是使用cookie作为媒介，存放用户凭证。
用户登录父应用之后，应用返回一个加密的cookie，当用户访问子应用的时候，携带上这个cookie，授权应用解密cookie并进行校验，校验通过则登录当前用户。

不难发现以上方式把信任存储在客户端的Cookie中，这种方式很容易令人质疑：

>- Cookie不安全
>- 不能跨域实现免登

对于第一个问题，通过加密Cookie可以保证安全性，当然这是在源代码不泄露的前提下。如果Cookie的加密算法泄露，攻击者通过伪造Cookie则可以伪造特定用户身份，这是很危险的。
对于第二个问题，更是硬伤。

#### 通过JSONP实现
对于跨域问题，可以使用JSONP实现。
用户在父应用中登录后，跟Session匹配的Cookie会存到客户端中，当用户需要登录子应用的时候，授权应用访问父应用提供的JSONP接口，并在请求中带上父应用域名下的Cookie，父应用接收到请求，验证用户的登录状态，返回加密的信息，子应用通过解析返回来的加密信息来验证用户，如果通过验证则登录用户。
![img](http://upload-images.jianshu.io/upload_images/79702-7ddba46df098374b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  通过页面重定向的方式

是通过父应用和子应用来回重定向中进行通信，实现信息的安全传递。
父应用提供一个GET方式的登录接口，用户通过子应用重定向连接的方式访问这个接口，如果用户还没有登录，则返回一个的登录页面，用户输入账号密码进行登录。如果用户已经登录了，则生成加密的Token，并且重定向到子应用提供的验证Token的接口，通过解密和校验之后，子应用登录当前用户。


### OAuth2.0登录

这就比较普遍了，现在随随便便做个网站，都接入「微信登录」、「微博登录」、「豆瓣登录」、「QQ登录」、「Github登录」,这些统一叫做：「第三方登录」。
第三方登录都是实现了OAuth2.0协议的，流程大概是酱紫的：
第三方提供一个登录入口，也就是第三方域名下的登录页面。主站需要登录的时候，引导用户重定向到第三方的登录页面，用户输入账号密码之后，登录第三方系统，第三方系统匹配帐号成功之后，带上一个code到主站的回调地址，主站接收到code，短时间内拿着code请求第三方提供获取长期凭证的接口(因为code有一个比较短的过期时间)，这个长期凭证叫`access_token`，获取之后就把这个`access_token`存到数据库中，请求一些第三方提供的API，需要用到这个`access_token`，因为这个token就是记录用户在第三方系统的一个身份凭证。
一些系统，在获取`access_token`的时候，还会返回一个副参数`refresh_token`，因为`access_token`是有过期时间的，一旦过期了，主站可以使用refresh_token请求第三方提供的接口获取新的`access_token`以及新的`refresh_token`。
在Nodejs中，你可以使用passport来给第三方登录提供一个统一解决方案，而如果你是开发「微信公众号」授权，除了[passport](https://www.npmjs.com/package/passport)，也可以使用[wechat-oauth](https://www.npmjs.com/package/wechat-oauth)


### 其他

其实登录问题，理解了Session原理是很重要的，这个也不难理解。然后站点之间的用户信息交流，就是通过各种跨域限制，各种加密解密而已。在做这个的时候，需要充分考虑到加密的token是否会被窃取的可能性，还要考虑让这个token加上时间的验证，在一些可能会被窃取，安全需求比较高的情况，就需要把token的时间设置的更短。

加密的方式需要依照需求不同而选择可逆或者不可逆，`hash sha1`,还是`JWT(Json Web Token)`。
sha1加密，可以使用Nodejs自带的`crypto`,php 的 `sha1 `，JWT可以使用[json web token](https://jwt.io/)

