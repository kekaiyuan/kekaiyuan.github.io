---
layout: post
title: 非对称加密
categories: [cate1, cate2]
description: some word here
keywords: keyword1, keyword2
mermaid: true
sequence: true
---

# 非对称加密

```sequence
title: 非对称加密流程
participant 客户端 as A
participant 服务器 as B
Note over A: 生成密钥对 A
A->B: 发送公钥 A
Note over B: 生成密钥对 B
B->A: 发送公钥 B
Note over A: 使用\n公钥 B 加密\n数据
A->B: 发送密文
Note over B: 使用\n私钥 B 解密\n数据
Note over B: 使用\n公钥 A 加密\n数据
B->A: 发送密文
Note over A: 使用\n私钥 A 解密\n数据
```
非对称加密的安全性极高。
- 在网络中传递的只有公钥，并没有传递私钥。
- 第三方即使获取了公钥，也是无法解开密文的，密文必须使用私钥解密。

但是非对称加密的效率极低，比对称加密慢**数千倍**。

所以往往**混合**使用
- 使用非对称加密传递对称加密的密钥
- 使用对称加密传递一般数据

## 中间人攻击
非对称加密看上去很美，但它真的就是完美无缺的了吗？

非对称加密依然可以被**破解**。
```sequence
title: 中间人攻击
participant 客户端 as A
participant 中间人 as C
participant 服务器 as B
participant B
Note over A: 生成密钥对 A
A->B: 发送公钥 A
Note over B: 生成密钥对 B
B->C: 发送公钥 B
Note over C: 截取公钥 B
Note over C: 生成密钥对 C
C->A: 发送公钥 C
Note over A: 使用\n公钥 C 加密\n数据
A->C: 发送密文
Note over C: 截取密文
Note over C: 使用\n私钥 C 解密\n数据
Note over C: 修改明文
Note over C: 使用\n公钥 B 加密\n修改后的明文
C->B: 发送修改后的密文
Note over B: 使用\n私钥 B 解密\n修改后的密文
Note over B: 执行错误操作
```

**真实案例**

> 南京90后小伙从网上学来了一种小技巧，只花1分钱就能成功充值上百元话费，结果因涉嫌盗窃罪被警方批捕。
> 
> 据央视新闻报道，小伙无业，在一个“薅羊毛”的聊天群里，于是“先是用1分钱充了200元成功到账，然后就开始反复充值，一共到账58笔、26000余元。”
> 
> 除了这名90后小伙，聊天群的其他数十位网友，仅仅两天就通过这个攻略，“薅”走了160多万元话费。
> 
> 经调查，江苏某电子商务公司的网上话费充值平台由于没有设置校验环节，导致支付环节存在技术漏洞，从而被黑客利用。
> 
> 日前，小刘等人因涉嫌盗窃罪，被南京市检雨花台区检察院正式批捕。

这就是中间人攻击。
- 原先
	- 甲通过平台充值 200 元话费，平台会根据甲提供的支付手段（银行卡/支付宝/微信钱包）扣除 200 元。
- 中间人攻击
	- 乙先通过平台充值 200 元话费，然后截取平台发送给支付方的消息，修改金额为 0.01 元。
	- 并且平台没有设置校验环节。
		- 我给用户充值了 200 元，但是我有没有扣除用户 200 元呢？

### 解决方法
中间人攻击的关键在哪儿？<br>
在于**客户端**无法分辨**服务器**和**中间人**。
- **服务器**说“我是服务器”。
- **中间人**也说“我是服务器”。

最终**客户端**使用了**中间人**的公钥来加密明文，使得明文被中间人**修改**了。

在现实生活中，如何分辨一个人呢？
- 要求他出示身份证，用机器扫描这张身份证是不是真的。
- 再人脸识别这个人和身份证是否匹配。

在网络中，也有类似于身份证的东西——**证书**。
- 最正规，最值得信赖的证书是由 **CA（数字证书颁发机构）** 颁发的。
	- 这是最权威的，在互联网类似于国家背书的身份证一样。
- 技术能力强的程序员可以**自己实现**证书。
	- 这个证书就没什么可信度了，类似于街头的小公司。
		- 有的小公司虽然小，但是是个正经公司。
		- 有的是皮包公司。
		- 有的是传销组织。
		- ......

对于一般用户来说，他们不需要担心自己的电脑（即客户端）合不合法，他们只担心访问的网站（服务器）合不合法。<br>
客户端发出请求后，服务器会在响应报文中放入自己的**数字证书**，以便客户端检查得到的消息是否是正确的。<br>
各类浏览器都会实现对于证书的解析，有问题的会进行如下提示：<br>
![image](\images\posts\encryption\asymmetric\unsafe.png)<br>
这就是证书有问题的网站：**它无法证明它是自己**。<br>
尽量不要使用此类网站，如果一定要使用，请注意：
- 谨慎填写任何有用的信息，包括但不限于
	- 个人信息
	- 任何账号密码
- 谨慎进行任何金钱操作。

#### 具体实现
我们已经知道了非对称加密和解决方法，接下来看一看具体实现。

通过仔细观察中间人攻击的流程，我们可以发现两个关键点：
1. 中间人把服务器的公钥换成了自己的。
2. 中间人修改了明文。

所以要防御中间人攻击，我们必须解决这两点。
1. 数字证书
2. 数字签名

##### 数字证书
数字证书中到底有什么呢？<br>
数字证书中存储了网站的各种相关信息（包括域名）。<br>
最重要的是，它存储了网站的**公钥**。

原本，我们通过网络直接传递公钥。<br>
但现在，我们通过数字证书传递公钥。

1. 服务器通过提交**各类信息**和自己的**公钥**向专业机构申请证书。
2. 专业机构在验证信息后，会形成证书。<br>
	**并且将证书使用自己的私钥加密（机构也有属于自己的密钥对）**。
3. 专业机构将加密后的证书交给服务器，服务器之后发送响应报文时会带上这张证书。
4. 客户端（浏览器）有一个**证书管理器**，其中有**受信任的根证书颁发机构**列表。<br>
	客户端收到响应报文后，会在列表中寻找能解开数字证书的**公钥**。<br>
	一般而言，会发生以下两种安全问题：
	- 证书的颁发机构不受信任。<br>
		![image](\images\posts\encryption\asymmetric\not-trusted.jpeg)
	- 证书的内容被修改了。<br>
		![image](\images\posts\encryption\asymmetric\be-modified.png)
5. 如果证书没问题，则客户端使用证书中记载的**服务器的公钥**解开报文。

##### 数字签名
数字签名又称为**摘要**。

为了防止正文不被修改，我们往往采用**数字签名**（又称为**摘要**）的方式。
- 将整篇正文做 Hash 运算，得到一个散列值（一般是 128 位）。
- 使用**发送方的私钥**将签名加密，附在正文后一起发送。
- 接收方收到数据后，先使用**发送方的公钥**将正文和签名都解开。
- 接收方再做一遍 Hash 运算，把结果和收到的签名进行比对。
	 - 一致：说明正文没有发生修改。
	- 不一致：说明正文被修改了。

这种方法是很难破解的：
- 不同的内容很难有一样的散列值。<br>
	`我是张三1`和`我是张三2`的散列值完全不一样。
- 即使散列值相同了，修改后的内容大概也是一团糟。<br>
	- `我是张三。`<br>
		`好的，你是张三。`
	- `$#^^*(**(&^%$#%@#%*&(*)))`<br>
		`不好意思，我听不懂。`
	- 此时接收方是无法识别的。

- 数字证书就采用了数字签名的方式保证自身的正确性。
	- 用机构的私钥加密。
	- 用机构的公钥解密。
- 很多网站在传递数据时也会使用数字签名。

