---
date: 2016-05-11T11:46:53+08:00
title: cookie&session
tags: ["http"]
---
> 因为http是无状态的协议，所以就需要有一套机制来保持用户的状态，比如用户的登录状态。这个机制其实就是借助cookie和session来完成的  

# cookie
Cookie用于存储http请求中的用户认证信息，用户在通过登录认证后，服务器在response header中设置Cookie，用户浏览器自动带Cookie访问该网站
通过在response 的header中添加set-cookie的信息来设置cookie，response中用set-cookie设置了cookie，那下次的request的header中就会带着这个cookie进行请求。cookie是一组key，value值。
```
Set-Cookie: <name>=<value>[; <name>=<value>]...
            [; expires=<date>][; domain=<domain_name>]
            [; path=<some_path>][; secure][; httponly]
```
- expires Cookie过期时间
- domain Cookie的生效域
- path Cookie的生效路径
- secure 表示cookie只能被发送到http服务器
- httponly 表示cookie不能被客户端脚本获取到

不带expires的Cookie表示临时Cookie只在当前会话有效，关闭浏览器Cookie实效

# session
session是保存在服务端的会话控制，一般服务器会发送包含sessionid的临时Cookie给浏览器，服务器根据sessionid获取保存的用户信息，自行维护过期时间等信息

拿登录请求为例，用户在登录页面输入用户名和密码，程序在后台验证用户的密码正确后，会按照一定的加密算法生成一个字符串，并通过在response的header中添加set-cookie来设置cookie，一般来说这个cookie的key是sessionid（当然名字是可以修改的不一定是sessionid），value就是加密后的字符串，这样在下次请求的header中就带有这样的一条cookie `sessionid=evt6iw35306xdzqu2xcdzfm8894h9cik`。程序后台会有一个表来存储session，表中一般会有session_key、 session_data、expire_date等字段。在下次请求时，程序就会拿到evt6iw35306xdzqu2xcdzfm8894h9cik，并查找session_key为evt6iw35306xdzqu2xcdzfm8894h9cik的session_data的值，进而解密出用户的信息，要是表中存在session_key为evt6iw35306xdzqu2xcdzfm8894h9cik的行，证明有这个用户，不存在就证明没有这个用户。
另外session也不一定是存在数据库里，存在memecache或者redis里甚至文件里都是可以的。
