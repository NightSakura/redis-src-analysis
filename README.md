# redis源码分析
基本上是顺着黄建宏老师的《Redis设计与实现》推荐的代码阅读顺序，再对着Redis的源码看看代码层面的具体的实现。

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
## 单机数据库的实现

## 多机数据库的实现

## 独立功能的实现
