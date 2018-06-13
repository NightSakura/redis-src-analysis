# Redis源码分析
随着对Redis的熟悉我想了解Redis实现等更多的内容，发现有很多现有的资料和方法指导，自然就站在巨人的肩膀上啦！

阅读顺序上基本上是顺着黄建宏老师的《Redis设计与实现》和[这篇博客](http://blog.huangz.me/diary/2014/how-to-read-redis-source-code.html)推荐的代码阅读顺序，另外也很感谢网易这位同学的[Redis解析系列](https://zcheng.ren/tags/Redis/page/2/)，就这样我对着源码分析看Redis的源码看看代码层面的具体的实现。坚持下来还是觉得收获颇丰，从服务器代码中去学习Reactor模式，I/O复用，设计模式等对之前学习的内容进行印证。

阅读的代码是基于黄建宏老师基于[Redis3.0的注释版本](https://github.com/huangz1990/redis-3.0-annotated)，之后有空会去读Redis5.0的代码。

## 数据结构与对象
* [内存分配](DataStructure/zmalloc.md)
* [简单动态字符串](DataStructure/sds.md)
* [链表](DataStructure/adlist.md)
* [字典](DataStructure/dict.md)
* [跳跃表](DataStructure/skiplist.md)
* [整数集合](DataStructure/intset.md)
* [压缩列表](DataStructure/zipset.md)
## 命令的实现
* [对象](DataStructure/robj.md)
* [字符串命令](Command/string.md)
* [列表命令](Command/list.md)
* [哈希命令](Command/hash.md)
* [集合命令](Command/set.md)
* [有序集合命令](Command/zset.md)
## 事件和网络处理
* [I/O复用](AE/io.md)
* [事件定义](AE/ae.md)
* [事件轮询](AE/eventLoop.md)
## 单机数据库的实现
* [](DB/db.md)

## 多机数据库的实现

## 独立功能的实现
