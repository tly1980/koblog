.. layout: post
.. title: "Email 与域名的配置 (SPF / DKIM)"
.. date: 2012-12-12 21:30
.. comments: true
.. categories: [技术, email, 域名]
.. keywords: email, DKIM, SPF, 邮件, mx, mta, 配置
.. description: 如何做邮件相关的域名配置。如何配置 SPF DKIM。


长久以来都积压了不少 email 跟域名的疑问。纸上得来终觉浅，最近终于有时间集中将这些东西捋一下，具体实战一下。理清一些思路，有了更好的理解。

### 第一点域名上的 MX 配置主要是用来收信的。

譬如某个仁兄(foo@example.com) 要往 abc@keyonly.com 发信，这个仁兄的 MTU（就是 Postfix 或者 Sendmail ）首先向 keyonly.com 的 域名服务器查询 keyonly.com 的 MX 记录。    
譬如查到 MX 记录指向 mail.keyonly.com ，example.com MTU 就会向 mail.keyonly.com 的 MTA 发送 SMTP 请求 （现在流行 ESMTP ）进行投递。

发 email 第一步解决了，知道信要往哪一个邮局发去，即往收件人的域名地址的 MX 记录上指向的机器发去。    
问题又来了，如何确认发信人的身份呢？

### 通过 SPF 跟 DKIM 确认发信人的身份
SMTP 协议允许任何计算机以任何源地址发邮件。因此有很多专门做垃圾邮件的人以伪装邮件的方式来发信。要解决这个问题，还是通过域名。

> 1. SPF 是用于验证发件人地址是否被某个域认可，即 **mailed-by**；    
> 2. DomainKey/DKIM 用来验证邮件内容是否被某个域认可，即 **signed-by**。

### 1.) SPF

SPF 实质是通过域名服务器向外中提供一个 Outbound MTA 白名单列表。

举一具体例子。

这些信息可以通过 dig 命令获取。

```
dig TXT keyonly.com
```

```
;; ANSWER SECTION:
keyonly.com.		86360	IN	TXT	"v=spf1 include:_spf.google.com ~all"
```
这是 keyonly.com 的配置，相对简单，仅仅允许 _spf.google.com，其他都会 softfail 掉 (~all)。 
如果配置为：

```
;; ANSWER SECTION:
keyonly.com.		86360	IN	TXT	"v=spf1 a mx include:_spf.google.com include:sendgrid.net ~all"
```

就是还允许 keyonly.com 的所有 a 记录跟 mx 记录，_spf.google.com 和 sendgrid.net 。

spf 配置，可以很灵活，很强大，详细请看这里：
http://www.openspf.org/SPF_Record_Syntax


### 2.) DKIM

DKIM 跟 DomainKey 有点大同小异，都是在 MTA （注意，这个是 Outbound MTA）发信时，对邮件进行签名。私钥由 Outbound MTA 保管着，不对外公开，公钥就放在域名的 TXT 项目上。接收方要验证信息的签名，可根据邮件上提供的信息，构造出含有公钥信息的域名地址，去该地址获取公钥。

对据说 DomainKey 是 Yahoo 发明的，是 DKIM 前身。貌似 DKIM 更强大，更流行。具体那个地方强大，请自己狗一下。而 Google app 也支持 DKIM。

注意，是签名，而不是加密。签名就是暗示着对邮件体的 Digest 通过私钥进行加密。Inbound MTA 在接收时候，就会从发件人域名服务器上查到公钥信息，然后对签名信息进行解密。如果解密得出的 Digest 值跟计算出来的一致，那证明邮件的合法性。


譬如我有某个从 keyonly.com 发出来的 email，其 DKIM 签名如下：

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
        d=keyonly.com; s=google;
        h=mime-version:x-originating-ip:date:message-id:subject:from:to
         :content-type;
        bh=EVanec6ZKH/vkbb95pMLG07OuPp+QtQ3z71oXKCTejw=;
        b=E02Fj/ROj18sLVc395zPdCTRu1L8Gsie63QG+tF4PoeS7z3Yzvn7m2zWArjQfaVN51
         hSLE7jXRm1S5y4uaZboO2jRy9stCeikZ5xs/38MrDZGh8IlkrUHySouHeqmdh5epwugT
         riTKjLMUF9t4YNO75i14mptgphw/7VNP2tjTY=
```


d=keyonly.com; s=google
重新构造出 DKIM 信息的域名：

```
${s}._domainkey.${d}
```

即：

```
google._domainkey.keyonly.com
```

所以我们可以通过 dig 获取 这个 DKIM 的公钥：

```
dig TXT google._domainkey.keyonly.com
```

```
google._domainkey.keyonly.com. 86400 IN	TXT	"v=DKIM1\; k=rsa\; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCF33Yl1sVLxLcS5UYUDdctOv2pdbaiYm1FRdSFzjvtO1b05zeXMJWKzXpGqpqh3i9sNNosrfmGKjjp/v+mklihVJUv7gRy/SyHg1WI8zRZNGfBtS0rE4s+jGeqtI2B2s4anJ0fcsps7N0kYjArBPCrv7LspPnCnHn6bggJZXjsGwIDAQAB"
```

一个域名上允许多个 DKIM 公钥。因为 DKIM 签名上的 s 项只要不重复就好了，所以我们完全可以有

* google._domain.keyonly.com  => google app 的 dkim
* sendgrid._domain.keyonly.com => sendgrid 的 dkim
* postfix._domain.keyonly.com  => 我的 postfix 的 dkim


###  3）SPF 跟 DKIM 的结合使用

SPF 跟 DKIM 完全可以绑定到不同域名上。譬如说，从 keyonly.com 发出来的 email 不用必须是用 keyonly.com 签名的，完全可以是其他域名签名的。

譬如说，我收到 NAB 银行的 email 签名的就是 Ubank (Ubank 是 NAB 的一个子银行)。感觉是他们	共用某些 IT 构架所导致的。

## 后记

* 配置的时候牵涉到 DNS 设置，可以通过 dig 命令来获取 DNS 信息。  
* 而 gmail 允许的一些特性会另调试变得简单：
	* signed by, mailed by （对着收件人按右键）。
	* show original 会显示邮件的原始信息。

一般大的 email 服务器会要求 SPF / DKIM 至少有一样；否则被 SPAM 几率就会增加。

实际上有很多大公司大机构，都不一定将这个问题处理得很好。譬如 NAB （National Australia Bank） 的 email 就连 SPF 都没有设置，虽然有设置 DKIM，但却用的是自己子银行 Ubank 的。严格来说，这不是一个金融公司专业的 IT 表现。实际上 NAB 前段还专门发信通知大家小心被钓鱼邮件骗。如果它能一早就做好 SPF / DKIM ，应该可以提高犯罪分子的作案难度，减少出现问题的机会。

而这点 CBA 就比 NAB 做得好多了，SPF / DKIM 都正确设置了，感觉比较专业。

