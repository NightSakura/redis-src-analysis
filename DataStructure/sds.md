# 简单动态字符串
C语言中并没有内置字符串，而是通过char类型的数组实现字符串的表示。字符串在Redis中的使用非常广泛，所有的key都是字符串类型，value中也有字符串类型。Redis为了实现二进制安全的字符串和更加强大的字符串功能等，自定义了字符串的数据结构。
## 自定义类型
 ```c
 struct sdshdr {
    // buf 中已占用空间的长度
    int len;
    // buf 中剩余可用空间的长度
    int free;
    // 数据空间
    char buf[];
};
 ```
 有关字符串的API包括如下所示的函数。
 ```c
 sds sdsnewlen(const void *init, size_t initlen);
sds sdsnew(const char *init);
sds sdsempty(void);
size_t sdslen(const sds s);
sds sdsdup(const sds s);
void sdsfree(sds s);
size_t sdsavail(const sds s);
sds sdsgrowzero(sds s, size_t len);
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);
sds sdscatsds(sds s, const sds t);
sds sdscpylen(sds s, const char *t, size_t len);
sds sdscpy(sds s, const char *t);
 ```
 ## 简单动态字符串的API分析
 ****
 ```c
 ```
 
  ****
 ```c
 ```
 
  ****
 ```c
 ```
 
  ****
 ```c
 ```
 
  ****
 ```c
 ```
 
  ****
 ```c
 ```
 
  ****
 ```c
 ```
 
