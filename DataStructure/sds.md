# 动态字符串
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
 Redis对字符串的封装的优点可以总结如下：
 * 二进制安全 C语言中的字符串遇到'\0'就认为到了字符串的结尾，并且字符串中只能存储符合编码规范的字符。而Redis用len表示字符串的长度，可以存储任何字符数据，并且Redis提供了二进制安全的api，所以SDS不仅可以存储文本数据，还可以存储任何格式的二进制数据。
 * O(1)时间复杂度获取字符串长度 
 * 降低了修改字符串进行的内存分配的次数，通过预分配空间和惰性的空间释放，降低了修改字符串需要重新分配内存的次数。
 * 杜绝缓冲区溢出问题，由于SDS内保存了剩余可用空间的长度，所以每次修改字符串时都会先检查空间再进行字符串修改。
 
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
 **sdsnewlen(const void *init, size_t initlen)**
 
 根据给定的初始化字符串init和字符串长度initlen创建一个新的 sds
 ```c
 sds sdsnewlen(const void *init, size_t initlen) {

    struct sdshdr *sh;
    // 根据是否有初始化内容，选择适当的内存分配方式
    // T = O(N)
    if (init) {
        // zmalloc 不初始化所分配的内存
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        // zcalloc 将分配的内存全部初始化为 0
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }

    // 内存分配失败，返回
    if (sh == NULL) return NULL;

    // 设置初始化长度
    sh->len = initlen;
    // 新 sds 不预留任何空间
    sh->free = 0;
    // 如果有指定初始化内容，将它们复制到 sdshdr 的 buf 中
    // T = O(N)
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    // 以 \0 结尾
    sh->buf[initlen] = '\0';

    // 返回 buf 部分，而不是整个 sdshdr
    return (char*)sh->buf;
}
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
 
