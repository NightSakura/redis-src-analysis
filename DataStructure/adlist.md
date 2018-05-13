# 链表
由于Redis使用的C语言没有内置链表的数据结构，因此Redis构建了自己的链表实现。该链表为双向链表，可以自定义存储类型，另外可以自定义释放、拷贝、对比函数等功能，提供的操作也比较丰富，包括创建、添加、插入、删除、索引等步骤，还拥有迭代器功能。
## 自定义类型
双向链表节点
```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
```
链表迭代器
```c
typedef struct listIter {
    // 当前迭代到的节点
    listNode *next;
    // 迭代的方向
    int direction;
} listIter;
```
链表
```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
```
Redis链表的特性可以总结如下：
* 双端链表，链表节点带有prev和next指针，获取某节点的前置节点和后置节点的复杂度为O（1）
* 带头尾指针和长度计数器，所以访问head和tail指针，链表中节点数量的时间复杂度为O（1）
* 多态，链表节点使用void* 来保存节点值，通过list结构的dup，free，match三个属性为节点设置类型特定函数，保存不同类型的值

## 链表和链表节点的API
**创建新的链表listCreate()**
```c
list *listCreate(void) {
    struct list *list;
    // 分配内存
    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    // 初始化属性
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    
    return list;
}
```

**释放链表listRelease()**
```c
void listRelease(list *list) {
    unsigned long len;
    listNode *current, *next;
    // 指向头指针
    current = list->head;
    // 遍历整个链表
    len = list->len;
    while(len--) {
        next = current->next;
        // 如果有设置值释放函数，那么调用它
        if (list->free) list->free(current->value);
        // 释放节点结构
        zfree(current);
        current = next;
    }
    // 释放链表结构
    zfree(list);
}
```

**添加链表节点到表头listAddNodeHead()**
```c
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;
    // 为节点分配内存
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    // 保存值指针
    node->value = value;
    // 添加节点到空链表
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    // 添加节点到非空链表
    } else {
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    // 更新链表节点数
    list->len++;

    return list;
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

****
```c
```

****
```c
```

****
```c
```


