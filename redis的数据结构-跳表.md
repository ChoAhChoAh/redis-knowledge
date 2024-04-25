# redis中的数据结构
## 跳表
### 一、结构
在最新unstable分支中，跳表的数据结构定义在server.h里。  
跳表用于组成zset结构，即有序集合。有序集合底层除了使用跳表外，还使用了字典（dict）,因此有序集合可以支持O(logn)+M的根据权重范围搜索，和O(1)的通过键获取权重。
跳表数据结构定义代码如下。   
在zskiplistNode跳表节点中，level数组中的每一个元素对应了一个zskiplistLevel结构体，对应了跳表的一层。  
每层的接头体中又包含了一个前向指针*forward,允许每层可以和后续指针连接。  
span用于记录该层前后两个节点相对level0横跨了几个节点的跨度。
总的来说，跳表可以归结为多层的有序链表。
```c
/* Zset使用一种特殊版本的跳表结构 */
typedef struct zskiplistNode {
    sds ele;  // Sorted Set中的元素
    double score; // 元素权重值
    struct zskiplistNode *backward; // 向后指针
    // 节点的level数组，保存每层上的前向指针和跨度
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 头尾节点
    unsigned long length; // 跳表长度
    int level;  // 层级
} zskiplist;
```
上述代码中，除跳表节点结构外，还定位了跳表结构zskiplist，包含头结点、尾结点、跳表长度和层级。
头尾节点用于将链表串联，用于通过指针访问跳表中的各个节点。

额外补充zset的结构定义，也在server.h中。
```c
typedef struct zset {
    dict *dict; // 字典结构(指针)
    zskiplist *zsl; // 跳表结构(指针)
} zset;
```

### 二、特性
1.跳跃查询（类似索引的b+树）：通过跳表的层级数组跳跃加速查询。查询一个节点时，会先从头结点的最高层查找下一个节点。  
（1）当查找的节点保存的元素权重小于要查找的权重，继续在当前层访问下一个节点。  
（2）当查找的节点权重相等时，检查ele元素，进行比较，如果sds小于要查找的元素，继续在当前层查找下一个节点。
不满足上述两个条件，进入level数组，使用下一层指针，跳跃层级继续查找。
2.随机生成每个节点的层数：redis跳表层级的设计并未按照上一层：下一层=1：2的比例进行设计，因为考虑到链表层级的调整，redis采用了节点随机设置层级的方式。每个节点仅有25%的概率会增加一层，且限制最大层数为64。
3.跳表和字典配合使用构造了zset结构：可以支持O(logn)+M的根据权重范围查找，和根据元素值ele进行O(1)的精确查找。

### 三、部分函数源码
#### 3.1 创建跳表函数

```c 
/* ZSKIPLIST_MAXLEVEL 定义了跳跃表的最大层数，这个值应该足够存储 2^64 个元素。
 * 跳跃表的层数越高，查找元素的速度越快，但是会占用更多的内存空间。 */
#define ZSKIPLIST_MAXLEVEL 32 /* 应该足够存储 2^64 个元素 */

/* 创建一个新的跳跃表。*/
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    // 分配内存给跳跃表结构
    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1; // 设置跳跃表初始层数为1
    zsl->length = 0; // 设置跳跃表长度为0
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL); // 创建跳跃表的头节点,最大层高32（ZSKIPLIST_MAXLEVEL）
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        // 初始化头节点的每一层
        zsl->header->level[j].forward = NULL; // 设置当前层的forward指针为NULL
        zsl->header->level[j].span = 0; // 设置当前层的span值为0
    }
    zsl->header->backward = NULL; // 设置头节点的backward指针为NULL
    zsl->tail = NULL; // 设置跳跃表尾节点为NULL
    return zsl; // 返回创建的跳跃表
}

/* 创建一个具有指定层数的跳跃表节点。
 * 在调用后，SDS字符串 'ele' 将由节点引用。 */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel)); // 分配内存给节点结构及其层级
    zn->score = score; // 设置节点的分值
    zn->ele = ele; // 设置节点的元素
    return zn; // 返回创建的节点
}
```

#### 3.2 添加元素函数

```c
/* 在一个有序集合中添加一个新元素或更新现有元素的分值，无论其编码方式如何。
 *
 * 下面是命令行为的一组标志。
 *
 * 输入标志如下：
 *
 * ZADD_INCR：将当前元素的分值增加 'score'，而不是更新当前元素的分值。
 *            如果元素不存在，则假定前一个分值为0。
 * ZADD_NX：   仅在元素不存在时执行操作。
 * ZADD_XX：   仅在元素已经存在时执行操作。
 * ZADD_GT：   仅在新分值大于当前分值时对现有元素执行操作。
 * ZADD_LT：   仅在新分值小于当前分值时对现有元素执行操作。
 *
 * 当使用 ZADD_INCR 时，如果 'newscore' 不为 NULL，则将元素的新分值存储在 '*newscore' 中。
 *
 * 返回的标志如下：
 *
 * ZADD_NAN：     结果分值不是一个数字。
 * ZADD_ADDED：   已添加元素（在调用之前不存在）。
 * ZADD_UPDATED： 已更新元素的分值。
 * ZADD_NOP：     由于 NX 或 XX 而未执行任何操作。
 *
 * 返回值：
 *
 * 函数在成功时返回1，并设置适当的标志（ADDED或UPDATED），以表示操作期间发生的情况（请注意，
 * 如果我们使用相同的分值重新添加元素，则可能不设置任何标志，或者在使用零增量的情况下）。
 *
 * 函数在出错时返回0，目前仅当增量产生 NaN 条件时，或者从开始时 'score' 值为 NaN 时。
 *
 * 作为添加新元素的副作用，命令可能会将有序集合的内部编码从 listpack 转换为 hashtable+skiplist。
 *
 * 'ele' 的内存管理：
 *
 * 函数不获取 'ele' SDS 字符串的所有权，但如果需要，则复制它。 */
int zsetAdd(robj *zobj,             // 有序集合对象
            double score,           // 元素的分值
            sds ele,               // 元素的值
            int in_flags,           // 输入标志
            int *out_flags,         // 输出标志
            double *newscore) {     // 新的分值（如果使用了 ZADD_INCR 标志）

    /* 将选项转换为易于检查的变量。*/
    int incr = (in_flags & ZADD_IN_INCR) != 0;   // 是否增量操作
    int nx = (in_flags & ZADD_IN_NX) != 0;       // 是否仅在元素不存在时执行
    int xx = (in_flags & ZADD_IN_XX) != 0;       // 是否仅在元素已存在时执行
    int gt = (in_flags & ZADD_IN_GT) != 0;       // 仅在新分值大于当前分值时执行
    int lt = (in_flags & ZADD_IN_LT) != 0;       // 仅在新分值小于当前分值时执行
    *out_flags = 0; /* 我们将返回响应的标志。*/
    double curscore;

    /* NaN 作为输入无论其他参数如何都是错误的。*/
    if (isnan(score)) {
        *out_flags = ZADD_OUT_NAN; // 设置标志为 NAN
        return 0; // 返回错误
    }

    /* 根据其编码更新有序集合。*/
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) {
        unsigned char *eptr;

        if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {
            /* NX？返回，相同元素已存在。*/
            if (nx) {
                *out_flags |= ZADD_OUT_NOP; // 设置标志为 NOP
                return 1; // 返回
            }

            /* 如果需要，为增量准备分值。*/
            if (incr) {
                score += curscore; // 增量操作，计算新的分值
                if (isnan(score)) {
                    *out_flags |= ZADD_OUT_NAN; // 设置标志为 NAN
                    return 0; // 返回错误
                }
            }

            /* GT/LT？仅在分值大于/小于当前分值时更新。*/
            if ((lt && score >= curscore) || (gt && score <= curscore)) {
                *out_flags |= ZADD_OUT_NOP; // 设置标志为 NOP
                return 1; // 返回
            }

            if (newscore) *newscore = score; // 存储新的分值

            /* 分值更改时删除并重新插入。*/
            if (score != curscore) {
                zobj->ptr = zzlDelete(zobj->ptr,eptr); // 删除原元素
                zobj->ptr = zzlInsert(zobj->ptr,ele,score); // 插入更新后的元素
                *out_flags |= ZADD_OUT_UPDATED; // 设置标志为 UPDATED
            }
            return 1; // 返回
        } else if (!xx) {
            /* 在执行 zzlInsert 之前检查元素是否过大或列表变得过长。*/
            if (zzlLength(zobj->ptr)+1 > server.zset_max_listpack_entries ||
                sdslen(ele) > server.zset_max_listpack_value ||
                !lpSafeToAdd(zobj->ptr, sdslen(ele)))
            {
                zsetConvertAndExpand(zobj, OBJ_ENCODING_SKIPLIST, zsetLength(zobj) + 1); // 将编码转换为 skiplist
            } else {
                zobj->ptr = zzlInsert(zobj->ptr,ele,score); // 插入元素
                if (newscore) *newscore = score; // 存储新的分值
                *out_flags |= ZADD_OUT_ADDED; // 设置标志为 ADDED
                return 1; // 返回
            }
        } else {
            *out_flags |= ZADD_OUT_NOP; // 设置标志为 NOP
            return 1; // 返回
        }
    }

    /* 注意，上面处理 listpack 的代码块要么返回要么将键转换为 skiplist。*/
    if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;

        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            /* NX？返回，相同元素已存在。*/
            if (nx) {
                *out_flags |= ZADD_OUT_NOP; // 设置标志为 NOP
                return 1; // 返回
            }

            curscore = *(double*)dictGetVal(de);

            /* 如果需要，为增量准备分值。*/
            if (incr) {
                score += curscore; // 增量操作，计算新的分值
                if (isnan(score)) {
                    *out_flags |= ZADD_OUT_NAN; // 设置标志为 NAN
                    return 0; // 返回错误
                }
            }

            /* GT/LT？仅在分值大于/小于当前分值时更新。*/
            if ((lt && score >= curscore) || (gt && score <= curscore)) {
                *out_flags |= ZADD_OUT_NOP; // 设置标志为 NOP
                return 1; // 返回
            }

            if (newscore) *newscore = score; // 存储新的分值

            /* 分值更改时删除并重新插入。*/
            if (score != curscore) {
                znode = zslUpdateScore(zs->zsl,curscore,ele,score); // 更新分值
                /* 注意，我们没有从表示有序集合的哈希表中删除原始元素，所以我们只更新分值。*/
                dictSetVal(zs->dict, de, &znode->score); /* 更新分值指针。*/
                *out_flags |= ZADD_OUT_UPDATED; // 设置标志为 UPDATED
            }
            return 1; // 返回
        } else if (!xx) {
            ele = sdsdup(ele);
            znode = zslInsert(zs->zsl,score,ele); // 插入新元素
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK); // 添加到字典中
            *out_flags |= ZADD_OUT_ADDED; // 设置标志为 ADDED
            if (newscore) *newscore = score; // 存储新的分值
            return 1; // 返回
        } else {
            *out_flags |= ZADD_OUT_NOP; // 设置标志为 NOP
            return 1; // 返回
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* 永远不会执行到这里。*/
}

/* 更新有序集合跳跃表中元素的分值。
 * 注意，元素必须存在并且必须匹配 'score'。
 * 此函数不更新哈希表中的分值，调用者应该注意。
 *
 * 注意，此函数试图仅更新节点，如果分值更新后，节点仍然处于完全相同的位置。
 * 否则，跳跃表将通过删除并重新添加一个新元素进行修改，这更加昂贵。
 *
 * 函数返回更新后的元素跳跃表节点指针。*/
zskiplistNode *zslUpdateScore(zskiplist *zsl,      // 有序集合跳跃表
                              double curscore,    // 当前分值
                              sds ele,           // 元素值
                              double newscore) { // 新分值
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    /* 我们需要定位到要更新的元素：无论如何，这都是有用的，
     * 我们将必须更新或删除它。*/
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    /* 跳到我们的元素：请注意，此函数假设具有匹配分值的元素存在。*/
    x = x->level[0].forward;
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    /* 如果节点在分值更新后仍然完全处于相同的位置，
     * 我们可以直接更新分值，而无需在跳跃表中实际删除并重新插入元素。*/
    if ((x->backward == NULL || x->backward->score < newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score > newscore))
    {
        x->score = newscore;
        return x;
    }

    /* 无法重用旧节点：我们需要在不同的位置删除并插入一个新节点。*/
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    /* 由于 zslInsert 创建了一个新节点，我们重用了旧节点 x->ele SDS 字符串，现在释放该节点。*/
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}



/* 在跳跃表中插入一个新节点。假设元素尚不存在（由调用者执行）。
 * 跳跃表接管传递的 SDS 字符串 'ele' 的所有权。*/
zskiplistNode *zslInsert(zskiplist *zsl,    // 跳跃表
                          double score,    // 分值
                          sds ele) {       // 元素值
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x; // 用于更新的节点数组以及当前节点指针
    unsigned long rank[ZSKIPLIST_MAXLEVEL]; // 每层跳跃表中插入位置的排名
    int i, level; // 循环变量以及插入节点的层数

    serverAssert(!isnan(score)); // 确保分值不是 NaN

    // 初始化当前节点指针为跳跃表头结点
    x = zsl->header;
    // 从顶层开始向下查找插入位置
    for (i = zsl->level-1; i >= 0; i--) {
        /* 存储到达插入位置所经过的排名 */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 向右查找插入位置
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span; // 更新排名
            x = x->level[i].forward; // 移动到下一个节点
        }
        update[i] = x; // 更新节点数组
    }

    /* 我们假设元素尚未存在，因为我们允许重复的分值，
     * 由于调用 zslInsert() 的函数应该在哈希表中测试元素是否已存在。
     * 所以重新插入相同元素不应该发生。*/
    // 随机生成节点的层数
    level = zslRandomLevel();
    if (level > zsl->level) {
        // 如果新节点的层数大于跳跃表当前的最大层数，需要更新层数和相关节点的 span 值
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0; // 新层的排名为 0
            update[i] = zsl->header; // 更新节点数组
            update[i]->level[i].span = zsl->length; // 设置新层的 span 值为跳跃表长度
        }
        zsl->level = level; // 更新跳跃表的最大层数
    }
    // 创建新节点
    x = zslCreateNode(level,score,ele);
    // 遍历每一层，将新节点插入到对应位置
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward; // 设置新节点的前向指针
        update[i]->level[i].forward = x; // 更新前向节点的指针为新节点

        /* 更新由 update[i] 插入 x 时覆盖的 span */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]); // 更新新节点的 span
        update[i]->level[i].span = (rank[0] - rank[i]) + 1; // 更新前向节点的 span
    }

    /* 未触及的级别增加 span */
    // 对于未触及的级别，只需要将前向节点的 span 增加 1 即可
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置新节点的后向指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    // 更新尾节点
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++; // 增加跳跃表的长度
    return x; // 返回新插入的节点
}
```

#### 3.3 查询元素函数

```c
/* 命令 ZRANGEBYSCORE <key> <min> <max> [WITHSCORES] [LIMIT offset count] */
void zrangebyscoreCommand(client *c) {
    // 初始化结果处理器
    zrange_result_handler handler;
	// 使用请求的类型初始化消费者接口类型
    zrangeResultHandlerInit(&handler, c, ZRANGE_CONSUMER_TYPE_CLIENT);
    // 调用通用的范围检索命令，向前检索
    zrangeGenericCommand(&handler, 1, 0, ZRANGE_SCORE, ZRANGE_DIRECTION_FORWARD);
}

/* 命令 ZREVRANGEBYSCORE <key> <max> <min> [WITHSCORES] [LIMIT offset count] */
void zrevrangebyscoreCommand(client *c) {
    // 初始化结果处理器
    zrange_result_handler handler;
	// 使用请求的类型初始化消费者接口类型
    zrangeResultHandlerInit(&handler, c, ZRANGE_CONSUMER_TYPE_CLIENT);
    // 调用通用的范围检索命令，向后检索
    zrangeGenericCommand(&handler, 1, 0, ZRANGE_SCORE, ZRANGE_DIRECTION_REVERSE);
}

/* 使用请求的类型初始化消费者接口类型。 */
static void zrangeResultHandlerInit(zrange_result_handler *handler,
    client *client, zrange_consumer_type type)
{
    memset(handler, 0, sizeof(*handler)); // 清空结果处理器

    handler->client = client; // 设置客户端指针

    // 根据类型选择相应的结果处理函数
    switch (type) {
    case ZRANGE_CONSUMER_TYPE_CLIENT: // 对于客户端消费类型
        handler->beginResultEmission = zrangeResultBeginClient; // 开始结果传输到客户端
        handler->finalizeResultEmission = zrangeResultFinalizeClient; // 结束结果传输到客户端
        handler->emitResultFromCBuffer = zrangeResultEmitCBufferToClient; // 从 C 缓冲区向客户端传输结果
        handler->emitResultFromLongLong = zrangeResultEmitLongLongToClient; // 从长长整型向客户端传输结果
        break;

    case ZRANGE_CONSUMER_TYPE_INTERNAL: // 对于内部消费类型
        handler->beginResultEmission = zrangeResultBeginStore; // 开始结果存储
        handler->finalizeResultEmission = zrangeResultFinalizeStore; // 结束结果存储
        handler->emitResultFromCBuffer = zrangeResultEmitCBufferForStore; // 从 C 缓冲区向存储传输结果
        handler->emitResultFromLongLong = zrangeResultEmitLongLongForStore; // 从长长整型向存储传输结果
        break;
    }
}

/* 该命令实现了 ZRANGE 和 ZREVRANGE。*/
void genericZrangebyrankCommand(zrange_result_handler *handler,
    robj *zobj, long start, long end, int withscores, int reverse) {

    client *c = handler->client; // 获取客户端
    long llen; // 有序集合的长度
    long rangelen; // 范围的长度
    size_t result_cardinality; // 结果基数（集合的元素数量）

    /* 校正索引。*/
    llen = zsetLength(zobj); // 获取有序集合的长度
    if (start < 0) start = llen + start; // 处理负索引
    if (end < 0) end = llen + end; // 处理负索引
    if (start < 0) start = 0; // 确保 start 不小于 0

    /* 不变性：start >= 0，因此当 end < 0 时此测试将为真。
     * 当 start > end 或 start >= length 时，范围为空。 */
    if (start > end || start >= llen) {
        handler->beginResultEmission(handler, 0); // 开始结果传输，结果基数为 0
        handler->finalizeResultEmission(handler, 0); // 结束结果传输，结果基数为 0
        return;
    }
    if (end >= llen) end = llen - 1; // 确保 end 不超过有序集合的长度
    rangelen = (end - start) + 1; // 计算范围的长度
    result_cardinality = rangelen; // 结果基数即为范围的长度

    handler->beginResultEmission(handler, rangelen); // 开始结果传输，结果基数为 rangelen
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) { // 如果是 ListPack 编码
        unsigned char *zl = zobj->ptr; // 获取有序集合的 ListPack
        unsigned char *eptr, *sptr; // 记录节点指针
        unsigned char *vstr; // 值的指针
        unsigned int vlen; // 值的长度
        long long vlong; // 值的长长整型形式
        double score = 0.0; // 分数

        if (reverse)
            eptr = lpSeek(zl, -2 - (2 * start)); // 向前查找节点
        else
            eptr = lpSeek(zl, 2 * start); // 向后查找节点

        serverAssertWithInfo(c, zobj, eptr != NULL); // 断言节点指针不为空
        sptr = lpNext(zl, eptr); // 获取值指针

        while (rangelen--) { // 遍历范围内的元素
            serverAssertWithInfo(c, zobj, eptr != NULL && sptr != NULL); // 断言节点指针和值指针均不为空
            vstr = lpGetValue(eptr, &vlen, &vlong); // 获取值

            if (withscores) /* 如果需要返回分数，则获取分数 */
                score = zzlGetScore(sptr);

            if (vstr == NULL) {
                handler->emitResultFromLongLong(handler, vlong, score); // 以长长整型形式传输结果
            } else {
                handler->emitResultFromCBuffer(handler, vstr, vlen, score); // 以缓冲区形式传输结果
            }

            if (reverse)
                zzlPrev(zl, &eptr, &sptr); // 向前移动指针
            else
                zzlNext(zl, &eptr, &sptr); // 向后移动指针
        }

    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) { // 如果是跳跃表编码
        zset *zs = zobj->ptr; // 获取有序集合结构
        zskiplist *zsl = zs->zsl; // 获取跳跃表
        zskiplistNode *ln;

        /* 检查起始点是否平凡，然后执行对数(N)查找。 */
        if (reverse) {
            ln = zsl->tail; // 如果是逆序，起始节点为尾节点
            if (start > 0)
                ln = zslGetElementByRank(zsl, llen - start); // 从尾部开始逆序查找
        } else {
            ln = zsl->header->level[0].forward; // 如果是顺序，起始节点为头节点的下一节点
            if (start > 0)
                ln = zslGetElementByRank(zsl, start + 1); // 从头部开始顺序查找
        }

        while (rangelen--) { // 遍历范围内的元素
            serverAssertWithInfo(c, zobj, ln != NULL); // 断言节点指针不为空
            sds ele = ln->ele; // 获取元素值
            handler->emitResultFromCBuffer(handler, ele, sdslen(ele), ln->score); // 传输结果
            ln = reverse ? ln->backward : ln->level[0].forward; // 根据方向选择下一个节点
        }
    } else {
        serverPanic("Unknown sorted set encoding"); // 未知的有序集合编码类型，抛出异常
    }

    handler->finalizeResultEmission(handler, result_cardinality); // 结束结果传输，传输结果基数
}

/* 根据排名查找元素。排名参数需要从 1 开始。*/
zskiplistNode *zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    return zslGetElementByRankFromNode(zsl->header, zsl->level - 1, rank); // 从头节点开始查找元素
}

/* 从起始节点开始根据排名查找元素。排名参数需要从 1 开始。*/
zskiplistNode *zslGetElementByRankFromNode(zskiplistNode *start_node, int start_level, unsigned long rank) {
    zskiplistNode *x; // 当前节点指针
    unsigned long traversed = 0; // 已遍历的距离
    int i;

    x = start_node; // 设置起始节点
    for (i = start_level; i >= 0; i--) { // 从高层往低层遍历
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            traversed += x->level[i].span; // 更新已遍历的距离
            x = x->level[i].forward; // 移动到下一个节点
        }
        if (traversed == rank) { // 如果已遍历的距离等于目标排名
            return x; // 返回找到的节点
        }
    }
    return NULL; // 没有找到满足条件的节点，返回空指针
}
```

#### 3.4 删除元素函数

```c
/* 从跳跃表中删除具有匹配分值/元素的节点。
 * 如果找到并删除了节点，则函数返回 1，否则返回 0。
 *
 * 如果 'node' 为 NULL，则删除的节点由 zslFreeNode() 释放，否则
 * 它不会被释放（只是取消链接），并将 *node 设置为节点指针，
 * 以便调用者可以重用节点（包括节点中引用的 SDS 字符串在 node->ele 中）。
 *
 * 参数：
 *   - zsl: 跳跃表指针。
 *   - score: 要匹配的节点分值。
 *   - ele: 要匹配的节点元素。
 *   - node: 可选参数，用于接收被删除的节点的指针。
 *
 * 返回值：
 *   - 找到并删除了节点时返回 1。
 *   - 未找到匹配节点时返回 0。
 */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x; // 更新节点数组、当前节点指针
    int i;

    x = zsl->header; // 设置当前节点为头节点
    // 从高层向低层遍历
    for (i = zsl->level-1; i >= 0; i--) {
        // 寻找要删除的节点的位置
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward; // 移动到下一个节点
        }
        update[i] = x; // 更新节点数组
    }
    // 检查找到的节点是否与要删除的节点匹配
    x = x->level[0].forward; // 移动到下一个节点
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) { // 如果找到了匹配的节点
        zslDeleteNode(zsl, x, update); // 从跳跃表中删除节点
        if (!node)
            zslFreeNode(x); // 释放节点
        else
            *node = x; // 设置节点指针
        return 1; // 返回找到并删除了节点
    }
    return 0; /* 未找到 */
}


/* 内部函数，被 zslDelete、zslDeleteRangeByScore 和 zslDeleteRangeByRank 调用，用于从跳跃表中删除节点。

   参数：
   - zsl: 跳跃表的指针。
   - x: 要删除的节点的指针。
   - update: 用于更新跳跃表的节点数组。

   这个函数的作用是从跳跃表中删除一个节点，并更新相关节点的指针和 span 值。

   实现细节：
   1. 遍历跳跃表的层级，更新每一层中被删除节点之后的节点的 span 值，以及前向指针。
   2. 更新被删除节点的前向指针，如果被删除节点是最后一个节点，则更新跳跃表的尾节点。
   3. 删除被删除节点之后，可能会导致跳跃表的层数减少，因此在循环中检查并更新跳跃表的层数。
   4. 减少跳跃表的长度。
*/
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    // 更新节点数组中被删除节点之后的节点的 span 值
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            // 如果被删除节点是下一个节点，则将 span 值减去被删除节点的 span，并更新前向指针
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            // 否则，只需要将 span 值减 1
            update[i]->level[i].span -= 1;
        }
    }
    // 更新被删除节点的前向指针
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        // 如果被删除节点是最后一个节点，则更新跳跃表的尾节点
        zsl->tail = x->backward;
    }
    // 删除被删除节点之后，可能会导致跳跃表的层数减少
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    // 减少跳跃表的长度
    zsl->length--;
}


/* 释放指定的跳跃表节点。除非在调用此函数之前将 node->ele 设置为 NULL，否则元素的 SDS 字符串表示也会被释放。

   参数：
   - node: 要释放的跳跃表节点的指针。

   这个函数的作用是释放跳跃表中的一个节点，同时也会释放节点中指向元素的 SDS 字符串表示。

   实现细节：
   1. 使用 sdsfree 函数释放节点的 ele 字段指向的 SDS 字符串。
   2. 使用 zfree 函数释放节点本身的内存空间。
*/
void zslFreeNode(zskiplistNode *node) {
    // 释放节点的 ele 字段指向的 SDS 字符串
    sdsfree(node->ele);
    // 释放节点本身的内存空间
    zfree(node);
}
```
### 四、应用
有序集合、redis内部集群节点