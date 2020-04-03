---
layout: post
title: mysql核心知识
date: 2019-06-05
Author: bkhech
tags: [mysql]
comments: true
toc: true
pinned: true
---

## mysql核心知识

先来思考以下几个问题：

1、索引的本质是什么？

2、三星索引有了解吗？ 你是如何理解的？

3、Innodb引擎如何保证事务的并发处理的？

4、你们公司SQL的执行最长的时间是多少秒？ 有分析过原因吗？ 解决的思路是怎样的？ 等等~

如果面对这些问题，这时你会一脸懵逼，没有完全正确的回答出来的小伙伴们可要反省反省了，毕竟这是目前一线互联网面试必问的知识点啊！

**MySQL有这么重要？我个人认为，MySQL掌握以下核心知识内容即可突破瓶颈**

1、掌握MySQL的整体体系结构，了解MySQL特色的各大存储引擎的特点。

2、深入MySQL的索引机制，做到每一个SQL执行能在脑海中构建数据搜索的过程。

3、理解MySQL中一条SQL语句的执行路径及每个环节的重要意义。 形成SQL执行的标准时序。

4、理解MySQL Innodb引擎的事务、锁、Redo/Undo、MVCC等机制。 充分理解Innodb引擎的优秀设计等等。