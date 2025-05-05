---
typora-root-url: D:\git\kekaiyuan.github.io
layout: post
title: template page
categories: [cate1, cate2]
description: some word here
keywords: keyword1, keyword2
---

在使用 JDBC 访问数据库时，需要建立连接和关闭连接。<br>
但是这两个操作都需要花费时间。

在高并发的环境中，为每一个操作都建立连接和关闭连接是不现实的。

于是就有了连接池的概念。

连接池中存放着多个连接，这些连接一般情况下是不会关闭的。

然后有 sql 操作时，从连接池中拿出一个连接，操作完毕把连接还给连接池。

这样 sql 操作就不需要单独地建立连接了。

连接池的设计一般要涉及以下四点：
1. 连接池的初始大小
2. 连接池的扩容<br>
3. 连接池的最大容量
4. 连接的死亡时间

## 源码链接
该文章源码链接 [Github](url)