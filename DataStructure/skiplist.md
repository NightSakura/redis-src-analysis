# 跳跃表
跳跃表的平均时间复杂度和平衡数相当，并且实现比较简单，Redis在实现中定制了这种数据类型。
## 自定义类型

跳跃表结点
```c
typedef struct zskiplistNode {
    // 成员对象
    robj *obj;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

跳跃表
```c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

## 跳跃表的API分析
**创建跳跃表结点zslCreateNode()**
```c
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    // 分配空间
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    // 设置属性
    zn->score = score;
    zn->obj = obj;
    return zn;
}
```

**创建新的跳跃表**
```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;
    // 分配空间
    zsl = zmalloc(sizeof(*zsl));
    // 设置高度和起始层数
    zsl->level = 1;
    zsl->length = 0;
    // 初始化表头节点
    // T = O(1)
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    // 设置表尾
    zsl->tail = NULL;
    return zsl;
}
```

**释放跳跃表结点**

减少跳跃表结点对应的对象的引用次数，然后释放内存
```c
void zslFreeNode(zskiplistNode *node) {
    decrRefCount(node->obj);
    zfree(node);
}
```

**释放跳跃表**

先释放表头，然后每次取level=0的前向指针依次释放所有的跳跃表结点
```c
void zslFree(zskiplist *zsl) {
    zskiplistNode *node = zsl->header->level[0].forward, *next;
    // 释放表头
    zfree(zsl->header);
    // 释放表中所有节点
    // T = O(N)
    while(node) {
        next = node->level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    // 释放跳跃表结构
    zfree(zsl);
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
