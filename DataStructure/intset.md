# 整数集合
我们知道Redis支持的数据结构保罗集合，而intset就是Redis用来保存整数值的集合抽象数据结构，它可以保存int16_t、int32_t、或者int64_t的整数值，并且保证集合中不会出现重复的元素。

## 自定义类型
```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

## 整数集合的API
```c
intset *intsetNew(void);
intset *intsetAdd(intset *is, int64_t value, uint8_t *success);
intset *intsetRemove(intset *is, int64_t value, int *success);
uint8_t intsetFind(intset *is, int64_t value);
int64_t intsetRandom(intset *is);
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value);
uint32_t intsetLen(intset *is);
size_t intsetBlobLen(intset *is);
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

****
```c
```
