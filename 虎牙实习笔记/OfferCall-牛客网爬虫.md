## 目标：

实现一个爬虫，爬取牛客的帖子

功能：

- 用户可以订阅有关键字的帖子，通过邮件、短信来接收新帖子的信息



## 爬虫框架：

SeimiCrawler

[中文文档](http://wiki.seimicrawler.org/#a1894e1210905b03f9b3404f8ca0b89)

[Github](https://github.com/zhegexiaohuozi/SeimiCrawler)

## 爬虫原理：

- 基本原理：

![基本原理](http://img.wanghaomiao.cn/v2_Seimi.png)

- 集群原理：
  ![集群原理](http://img.wanghaomiao.cn/v1_distributed.png)



## 难点：

1.爬取规则：
2.爬取内容解析

3.发布订阅模式的实现（最主要）：

- 实现考虑使用Redis的发布订阅模式+mysql来实现

-  不使用RabbitMQ的原因是订阅的时候动态创建exchange、queue以及消费者很麻烦

JSoupXPath



## 可优化方向：

- httpclient或OkHttp修改成netty支持的客户端Spring5的WebClient:
  - 相关类：SeimiProcessor类

- 分布式
- 代理ip



## 附加知识点

框架 Redisson



## 开工计划

- 7月2号：阅读SeimiCrawler的demo、源码，完成爬取牛客帖子的demo
- 3号：爬取策略的实现，策略模式的使用；
- 4号：定时任务，爬取新文章
- 5号：用户关键字订阅功能
- 6号：代理ip的实现
- 7、8号：WebClient的实现；分布式的实现

