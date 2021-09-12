---
title: 数据库是怎么工作的？
---

- 数据是以怎样的格式存储在内存和磁盘中的？
- 它们又是什么时候从内存持久化到磁盘的？
- 为什么一张表只能有一个主键？
- 怎样回滚一个事务？
- 索引格式是怎样的？
- 什么场景会触发全表扫描？
- 预处理语句的储存格式是什么样的？

简而言之，数据库是怎么**工作**的？

为了理解这些内容，我正在从零开始编写一个 [sqlite](https://www.sqlite.org/arch.html)，并且我将会用文字将做这些事的过程记录下来。

# 目录
{% for part in site.parts %}- [{{part.title}}]({{site.baseurl}}{{part.url}})
{% endfor %}

> "What I cannot create, I do not understand." -- [Richard Feynman](https://en.m.wikiquote.org/wiki/Richard_Feynman)  
> 译者注：「纸上得来终觉浅，绝知此事要躬行」 —— 陆游

{% include image.html url="assets/images/arch2.gif" description="sqlite architecture (https://www.sqlite.org/arch.html)" %}