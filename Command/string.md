# 字符串命令
Redis对用户可见的数据结构包括string、list、hash、set、zset5种，并且对每种类型都提供了很多操作命令。通过之前的分析我们已经知晓Redis的基础数据结构的表示，
从本节开始介绍Redis基于定制的数据结构和对象实现命令的具体方式。

在`redis.h`中维护了一个命令表redisCommandTable[]，实现了命令和方法的映射。命名规则的约定是方法名称=命令名称+Command。

## 字符串命令
首先，有必要复习下字符串相关的命令及其实现的功能，主要的字符串命令如下表所示。

|  命令  |  说明   |  
|:-----:|:-------:|
|SET key value [EX seconds] [PX milliseconds] [NX/XX]|将值value关联到key，支持参数形式|
|SETEX key seconds value|将值value关联到key，并设置生存时间seconds，如果key已存在覆写旧值|
|SETNX key value|将值value关联到key，当且仅当key不存在|
|STRLEN key|返回key所存储的字符串值的长度，如果key存储的不是字符串返回一个错误|
|GET key|返回key所关联的字符串值，如果key不存在返回特殊值nil，如果key存储的不是字符串返回一个错误|
|GETSET key value|将key的值设置为value，并返回key的旧值，当key存在且值不是字符串返回一个错误|
|MGET key [key...]|返回一个或者多个给定key的值|
|MSET key value [key value...]|同时设置多个key-value对，是原子性操作|
|MSETEX key value [key value...]|同时设置多个key-value对，当且仅当所有key都不存在|
|INCR key|将key中存储的数字值增一|
|INCRBY key increment|将key中存储的数字增加增量increment|
|DECR key |将key中存储的数字值减一|
|DECRBY key decrement|将key中存储的数字减少减量decrement|
|APPEND key value|将value追加到key原来的值末尾，如果key不存在等价于SET|

## 命令的实现
set命令的实现方法是setCommand(),此方法主要完成了选项参数的初始化、选项参数的解析和对值的对象化编码，而具体的执行操作交由setGenericCommand()来完成。
```c
/* SET key value [EX <seconds>] [PX <milliseconds>] [NX/XX]*/
void setCommand(redisClient *c) {
    int j;
    robj *expire = NULL;
    int unit = UNIT_SECONDS;
    int flags = REDIS_SET_NO_FLAGS;

    // 设置选项参数
    for (j = 3; j < c->argc; j++) {
        char *a = c->argv[j]->ptr;
        robj *next = (j == c->argc-1) ? NULL : c->argv[j+1];

        if ((a[0] == 'n' || a[0] == 'N') &&
            (a[1] == 'x' || a[1] == 'X') && a[2] == '\0') {
            flags |= REDIS_SET_NX;
        } else if ((a[0] == 'x' || a[0] == 'X') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0') {
            flags |= REDIS_SET_XX;
        } else if ((a[0] == 'e' || a[0] == 'E') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' && next) {
            unit = UNIT_SECONDS;
            expire = next;
            j++;
        } else if ((a[0] == 'p' || a[0] == 'P') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' && next) {
            unit = UNIT_MILLISECONDS;
            expire = next;
            j++;
        } else {
            addReply(c,shared.syntaxerr);
            return;
        }
    }

    // 尝试对值对象进行编码
    c->argv[2] = tryObjectEncoding(c->argv[2]);

    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}
```

flags 参数的值可以是 NX 或 XX ，它们
```c
void setGenericCommand(redisClient *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {

    long long milliseconds = 0; /* initialized to avoid any harmness warning */

    // 取出过期时间
    if (expire) {

        // 取出 expire 参数的值
        // T = O(N)
        if (getLongLongFromObjectOrReply(c, expire, &milliseconds, NULL) != REDIS_OK)
            return;

        // expire 参数的值不正确时报错
        if (milliseconds <= 0) {
            addReplyError(c,"invalid expire time in SETEX");
            return;
        }

        // 不论输入的过期时间是秒还是毫秒
        // Redis 实际都以毫秒的形式保存过期时间
        // 如果输入的过期时间为秒，那么将它转换为毫秒
        if (unit == UNIT_SECONDS) milliseconds *= 1000;
    }

    // 如果设置了 NX 或者 XX 参数，那么检查条件是否不符合这两个设置
    // 在条件不符合时报错，报错的内容由 abort_reply 参数决定
    if ((flags & REDIS_SET_NX && lookupKeyWrite(c->db,key) != NULL) ||
        (flags & REDIS_SET_XX && lookupKeyWrite(c->db,key) == NULL))
    {
        addReply(c, abort_reply ? abort_reply : shared.nullbulk);
        return;
    }

    // 将键值关联到数据库
    setKey(c->db,key,val);

    // 将数据库设为脏
    server.dirty++;

    // 为键设置过期时间
    if (expire) setExpire(c->db,key,mstime()+milliseconds);

    // 发送事件通知
    notifyKeyspaceEvent(REDIS_NOTIFY_STRING,"set",key,c->db->id);

    // 发送事件通知
    if (expire) notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,
        "expire",key,c->db->id);

    // 设置成功，向客户端发送回复
    // 回复的内容由 ok_reply 决定
    addReply(c, ok_reply ? ok_reply : shared.ok);
}
```
