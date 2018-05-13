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

创建成功返回list，创建失败返回NULL，时间复杂度O(1)
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

释放整个链表及所有链表节点，时间复杂度O(N)
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

如果为新节点分配内存出错，那么不执行任何动作，仅返回 NULLl；如果执行成功，返回传入的链表指针
头插法需要判断当前链表是不是空链表，如果是，设置该节点为list->head；如果不是设置list->head->prev为插入节点
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

**添加链表节点到表尾listAddNodeTail()**

如果为新节点分配内存出错，那么不执行任何动作，仅返回 NULLl；如果执行成功，返回传入的链表指针
```c
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;
    // 为新节点分配内存
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    // 保存值指针
    node->value = value;
    // 目标链表为空
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    // 目标链表非空
    } else {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    // 更新链表节点数
    list->len++;

    return list;
}
```

**创建链表节点并插入链表listInsertNode()**

创建一个包含值 value 的新节点，并将它插入到 old_node 的之前或之后
如果 after 为 0 ，将新节点插入到 old_node 之前。
如果 after 为 1 ，将新节点插入到 old_node 之后。
```c
list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
    listNode *node;
    // 创建新节点
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    // 保存值
    node->value = value;
    // 将新节点添加到给定节点之后
    if (after) {
        node->prev = old_node;
        node->next = old_node->next;
        // 给定节点是原表尾节点
        if (list->tail == old_node) {
            list->tail = node;
        }
    // 将新节点添加到给定节点之前
    } else {
        node->next = old_node;
        node->prev = old_node->prev;
        // 给定节点是原表头节点
        if (list->head == old_node) {
            list->head = node;
        }
    }
    // 更新新节点的前置指针
    if (node->prev != NULL) {
        node->prev->next = node;
    }
    // 更新新节点的后置指针
    if (node->next != NULL) {
        node->next->prev = node;
    }
    // 更新链表节点数
    list->len++;

    return list;
}
```

**删除链表节点listDelNode()**
```c
void listDelNode(list *list, listNode *node)
{
    // 调整前置节点的指针
    if (node->prev)
        node->prev->next = node->next;
    else
        list->head = node->next;
    // 调整后置节点的指针
    if (node->next)
        node->next->prev = node->prev;
    else
        list->tail = node->prev;
    // 释放值
    if (list->free) list->free(node->value);

    // 释放节点
    zfree(node);
    // 链表数减一
    list->len--;
}
```

**链表创建迭代器listGetIterator()**

为给定链表创建一个迭代器，之后每次对这个迭代器调用listNext,都返回被迭代到的链表节点
direction 参数决定了迭代器的迭代方向：AL_START_HEAD ：从表头向表尾迭代；AL_START_TAIL ：从表尾想表头迭代
时间复杂度为T = O(1)
```c
listIter *listGetIterator(list *list, int direction)
{
    // 为迭代器分配内存
    listIter *iter;
    if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;

    // 根据迭代方向，设置迭代器的起始节点
    if (direction == AL_START_HEAD)
        iter->next = list->head;
    else
        iter->next = list->tail;

    // 记录迭代方向
    iter->direction = direction;

    return iter;
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


