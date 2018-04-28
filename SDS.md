# 简单动态字符串SDS

Redis使用可自定义的字符串作为默认的字符串表示，其数据结构定义如下所示。
```c
struct sdshdr {
    // buf 中已占用空间的长度=SDS所保存字符串的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 字节数组，保存字符串的数据空间
    char buf[];
};
```
