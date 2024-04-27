# redis中的数据结构
## 快表（quicklist）
### 一、结构
quicklist是有由链表+listpack组结构成的数据结构，在6.0及之前的结构中，quicklist由链表+ziplist组成。  
由于ziplizt本身存在的问题，后被listpack取代。  
quicklist用于redis的list结构。
```c
/* Node、quicklist 和 Iterator 是当前使用的唯一数据结构。*/

/* quicklistNode 是一个 32 字节的结构体，用于描述 quicklist 的 listpack。
 * 我们使用位域来确保 quicklistNode 大小为 32 字节。
 * count: 16 位，最大 65536（listpack 最大字节数为 65k，因此最大 count 实际上小于 32k）。
 * encoding: 2 位，RAW=1（原始数据），LZF=2（LZF 压缩）。
 * container: 2 位，PLAIN=1（单个元素作为 char 数组），PACKED=2（listpack 包含多个元素）。
 * recompress: 1 位，布尔值，如果节点是临时解压缩的，则为 true。
 * attempted_compress: 1 位，布尔值，用于测试时验证。
 * dont_compress: 1 位，布尔值，用于阻止需要稍后使用的条目的压缩。
 * extra: 9 位，用于未来扩展的空闲位；填充了剩余的 32 位 */
typedef struct quicklistNode {
    struct quicklistNode *prev; // 指向前一个 quicklistNode 的指针
    struct quicklistNode *next; // 指向后一个 quicklistNode 的指针
    unsigned char *entry;       // 指向 listpack 的指针
    size_t sz;                  /* entry 的大小（字节数）*/
    unsigned int count : 16;    /* listpack 中条目的计数 */
    unsigned int encoding : 2;  /* 编码方式：RAW=1 或 LZF=2 */
    unsigned int container : 2; /* 容器类型：PLAIN=1 或 PACKED=2 */
    unsigned int recompress : 1; /* 前一个节点是否被压缩过 */
    unsigned int attempted_compress : 1; /* 节点无法压缩；太小 */
    unsigned int dont_compress : 1; /* 阻止需要稍后使用的条目的压缩 */
    unsigned int extra : 9; /* 未来用于扩展的额外位 */
} quicklistNode;

/* quicklistLZF 是一个 8+N 字节的结构体，包含 'sz' 和 'compressed'。
 * 'sz' 是 'compressed' 字段的字节长度。
 * 'compressed' 是 LZF 数据，总长度为 'sz'
 * 注意：未压缩的长度存储在 quicklistNode->sz 中。
 * 当 quicklistNode->entry 被压缩时，node->entry 指向 quicklistLZF */
typedef struct quicklistLZF {
    size_t sz; /* LZF 的字节大小 */
    char compressed[];
} quicklistLZF;

/* Bookmarks 在 quicklist 结构体末尾进行 realloc 扩展。
 * 仅在非常大的列表中使用，如果有数千个节点，多余的内存使用量可以忽略不计，
 * 并且确实需要分部分迭代它们时使用。
 * 当不使用时，它们不会增加任何内存开销，但是当使用后删除时，会留下一些开销（以避免共振）。
 * 使用的书签数量应保持最少，因为它还会增加节点删除时的开销（搜索要更新的书签）。*/
typedef struct quicklistBookmark {
    quicklistNode *node; // 指向 quicklistNode 的指针
    char *name;          // 书签名称
} quicklistBookmark;

```

### 二、特性


### 三、部分函数代码
#### 3.1 头部添加元素
```c
/* 在 quicklist 的头节点中添加新条目。
 *
 * 如果使用现有头节点，则返回 0。
 * 如果创建了新头节点，则返回 1。 */
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head; // 记录原始头节点

    if (unlikely(isLargeElement(sz, quicklist->fill))) { // 如果新条目过大
        __quicklistInsertPlainNode(quicklist, quicklist->head, value, sz, 0); // 将新节点插入为普通节点
        return 1;
    }

    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) { // 如果当前头节点还有足够空间插入新条目
        quicklist->head->entry = lpPrepend(quicklist->head->entry, value, sz); // 在原头节点的 entry 前插入新条目
        quicklistNodeUpdateSz(quicklist->head); // 更新头节点的大小信息
    } else { // 如果当前头节点没有足够空间插入新条目
        quicklistNode *node = quicklistCreateNode(); // 创建一个新节点
        node->entry = lpPrepend(lpNew(0), value, sz); // 在新节点的 entry 中插入新条目

        quicklistNodeUpdateSz(node); // 更新新节点的大小信息
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node); // 在当前头节点之前插入新节点
    }
    quicklist->count++; // 更新 quicklist 的条目总数
    quicklist->head->count++; // 更新头节点的条目数
    return (orig_head != quicklist->head); // 返回是否创建了新头节点
}

/* 将普通节点插入到 quicklist 中的内部函数。
 * 参数说明：
 * - quicklist: 要插入节点的 quicklist 结构体指针
 * - old_node: 要在其之后或之前插入新节点的节点指针
 * - value: 要插入的值的指针
 * - sz: 要插入的值的大小
 * - after: 插入位置，如果为 1，则在 old_node 之后插入；如果为 0，则在 old_node 之前插入 */
static void __quicklistInsertPlainNode(quicklist *quicklist, quicklistNode *old_node,
                                       void *value, size_t sz, int after)
{
    quicklistNode *new_node = __quicklistCreateNode(QUICKLIST_NODE_CONTAINER_PLAIN, value, sz); // 创建新的普通节点
    __quicklistInsertNode(quicklist, old_node, new_node, after); // 插入新节点到 quicklist
    quicklist->count++; // 更新 quicklist 的条目总数
}

/* 创建一个新的 quicklistNode 节点。
 * 参数说明：
 * - container: 节点的容器类型，可以是 QUICKLIST_NODE_CONTAINER_PLAIN 或 QUICKLIST_NODE_CONTAINER_PACKED
 * - value: 要插入节点的值的指针
 * - sz: 要插入节点的值的大小
 * 返回值：
 * - 返回新创建的 quicklistNode 节点指针。 */
static quicklistNode* __quicklistCreateNode(int container, void *value, size_t sz) {
    quicklistNode *new_node = quicklistCreateNode(); // 创建一个新的 quicklistNode 节点
    new_node->container = container; // 设置节点的容器类型

    if (container == QUICKLIST_NODE_CONTAINER_PLAIN) { // 如果节点的容器类型是普通容器
        new_node->entry = zmalloc(sz); // 分配内存给节点的 entry
        memcpy(new_node->entry, value, sz); // 将值复制到节点的 entry 中
    } else { // 否则，节点的容器类型是压缩容器
        new_node->entry = lpPrepend(lpNew(0), value, sz); // 创建 listpack，并将值插入到 listpack 的开头
    }
    new_node->sz = sz; // 设置节点的大小
    new_node->count++; // 增加节点的条目数
    return new_node; // 返回新创建的 quicklistNode 节点指针
}

/* 在 old_node 之后插入 new_node（如果 after 为 1）。
 * 在 old_node 之前插入 new_node（如果 after 为 0）。
 * 注意：new_node 始终是未压缩的，因此如果将其分配给 head 或 tail，则无需解压缩。 */
REDIS_STATIC void __quicklistInsertNode(quicklist *quicklist,
                                        quicklistNode *old_node,
                                        quicklistNode *new_node, int after) {
    if (after) { // 如果在 old_node 之后插入 new_node
        new_node->prev = old_node; // 设置 new_node 的前驱为 old_node
        if (old_node) { // 如果 old_node 不为空
            new_node->next = old_node->next; // 设置 new_node 的后继为 old_node 的后继
            if (old_node->next)
                old_node->next->prev = new_node; // 将 old_node 的后继的前驱设置为 new_node
            old_node->next = new_node; // 将 old_node 的后继设置为 new_node
        }
        if (quicklist->tail == old_node)
            quicklist->tail = new_node; // 如果 old_node 是尾节点，则更新 quicklist 的尾节点为 new_node
    } else { // 如果在 old_node 之前插入 new_node
        new_node->next = old_node; // 设置 new_node 的后继为 old_node
        if (old_node) { // 如果 old_node 不为空
            new_node->prev = old_node->prev; // 设置 new_node 的前驱为 old_node 的前驱
            if (old_node->prev)
                old_node->prev->next = new_node; // 将 old_node 的前驱的后继设置为 new_node
            old_node->prev = new_node; // 将 old_node 的前驱设置为 new_node
        }
        if (quicklist->head == old_node)
            quicklist->head = new_node; // 如果 old_node 是头节点，则更新 quicklist 的头节点为 new_node
    }

    /* 如果此插入操作是唯一的元素，初始化 head/tail。 */
    if (quicklist->len == 0) {
        quicklist->head = quicklist->tail = new_node; // 如果 quicklist 中没有元素，则将 head 和 tail 设置为 new_node
    }

    /* 先更新 len，这样在 __quicklistCompress 中我们就知道 len 的确切值 */
    quicklist->len++; // 更新 quicklist 的元素数量

    if (old_node)
        quicklistCompress(quicklist, old_node); // 如果 old_node 不为空，则压缩 quicklist

    quicklistCompress(quicklist, new_node); // 压缩 quicklist 中的 new_node
}



/* 检查是否可以在给定节点上插入新元素。
 * 参数说明：
 * - node: 要检查的节点指针
 * - fill: 填充限制，用于判断是否按条目数限制还是按大小限制
 * - sz: 要插入的新元素的大小
 * 返回值：
 * - 如果节点为空或者节点是普通节点或者新元素过大，则返回 0；
 * - 否则返回 1，表示可以在节点上插入新元素。 */
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz) {
    if (unlikely(!node)) // 检查节点是否为空
        return 0;

    if (unlikely(QL_NODE_IS_PLAIN(node) || isLargeElement(sz, fill))) // 检查节点是否是普通节点或者新元素是否过大
        return 0;

    /* 估算插入该新元素后 listpack 将增加的字节数。
     * 我们更倾向于过度估计，这样至少能确保不会低于 4k 的最低限制（参见 optimization_level）。
     * 注意：在上面的普通/大元素检查后，`node->sz` 和 `sz` 都应该小于 1GB，因此不需要检查溢出。 */
    size_t new_sz = node->sz + sz + SIZE_ESTIMATE_OVERHEAD; // 计算插入新元素后 listpack 将增加的字节数
    if (unlikely(quicklistNodeExceedsLimit(fill, new_sz, node->count + 1))) // 检查新节点是否超过了限制
        return 0;
    return 1;
}

```
#### 3.2 尾部添加元素

```c
/* 在 quicklist 的尾节点中添加新条目。
 *
 * 如果使用现有尾节点，则返回 0。
 * 如果创建了新尾节点，则返回 1。 */
int quicklistPushTail(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_tail = quicklist->tail; // 记录原始尾节点

    if (unlikely(isLargeElement(sz, quicklist->fill))) { // 如果新条目过大
        __quicklistInsertPlainNode(quicklist, quicklist->tail, value, sz, 1); // 将新节点插入为普通节点
        return 1;
    }

    if (likely(
            _quicklistNodeAllowInsert(quicklist->tail, quicklist->fill, sz))) { // 如果当前尾节点还有足够空间插入新条目
        quicklist->tail->entry = lpAppend(quicklist->tail->entry, value, sz); // 在原尾节点的 entry 后插入新条目
        quicklistNodeUpdateSz(quicklist->tail); // 更新尾节点的大小信息
    } else { // 如果当前尾节点没有足够空间插入新条目
        quicklistNode *node = quicklistCreateNode(); // 创建一个新节点
        node->entry = lpAppend(lpNew(0), value, sz); // 在新节点的 entry 中插入新条目

        quicklistNodeUpdateSz(node); // 更新新节点的大小信息
        _quicklistInsertNodeAfter(quicklist, quicklist->tail, node); // 在当前尾节点之后插入新节点
    }
    quicklist->count++; // 更新 quicklist 的条目总数
    quicklist->tail->count++; // 更新尾节点的条目数
    return (orig_tail != quicklist->tail); // 返回是否创建了新尾节点
}

```
#### 3.3 判断元素是否过大

```c
/* 多元素 listpack 的最大字节大小。
 * 更大的值将存在于它们自己独立的 listpack 中。
 * 仅当我们受记录计数限制时才会使用此值。当我们受大小限制时，最大限制更大，但仍然安全。
 * 推荐/默认大小限制为 8k */
#define SIZE_SAFETY_LIMIT 8192

#define sizeMeetsSafetyLimit(sz) ((sz) <= SIZE_SAFETY_LIMIT)
/* 根据 'fill' 确定给定大小是否符合大元素的条件。
 * 如果大小被视为大元素，它将存储在普通节点中。 */
static int isLargeElement(size_t sz, int fill) {
    if (unlikely(packed_threshold != 0)) return sz >= packed_threshold; // 如果 packed_threshold 不为 0，则直接比较大小与 packed_threshold 的关系
    if (fill >= 0)
        return !sizeMeetsSafetyLimit(sz); // 如果 fill 大于等于 0，则判断大小是否超过安全限制
    else
        return sz > quicklistNodeNegFillLimit(fill); // 否则，判断大小是否超过 quicklistNodeNegFillLimit 的负值
}

/* 基于负 'fill' 计算 quicklist 节点的大小限制。 */
static size_t quicklistNodeNegFillLimit(int fill) {
    assert(fill < 0); // 断言 fill 必须小于 0

    size_t offset = (-fill) - 1; // 计算 offset，即 (-fill) 减去 1
    size_t max_level = sizeof(optimization_level) / sizeof(*optimization_level); // 计算 optimization_level 数组的元素个数
    if (offset >= max_level) offset = max_level - 1; // 如果 offset 大于等于 max_level，则将 offset 设置为 max_level - 1
    return optimization_level[offset]; // 返回根据 offset 计算得到的 quicklist 节点的大小限制
}
```
#### 3.4 获取元素位置的迭代器

```c
/* 在特定偏移量 'idx' 处初始化迭代器，并使迭代器按 'direction' 方向返回节点。 */
quicklistIter *quicklistGetIteratorAtIdx(quicklist *quicklist,
                                         const int direction,
                                         const long long idx)
{
    quicklistNode *n; // 用于存储迭代器当前指向的节点
    unsigned long long accum = 0; // 用于累积计算偏移量
    unsigned long long index; // 迭代器当前的偏移量
    int forward = idx < 0 ? 0 : 1; /* < 0 -> reverse, 0+ -> forward */

    index = forward ? idx : (-idx) - 1; // 根据方向设置迭代器的初始偏移量
    if (index >= quicklist->count) // 如果偏移量超出了 quicklist 的范围，则返回空迭代器
        return NULL;

    /* 如果另一方向的距离更短，则在另一方向上寻找。 */
    int seek_forward = forward; // 是否向前寻找
    unsigned long long seek_index = index; // 寻找的目标偏移量
    if (index > (quicklist->count - 1) / 2) { // 如果偏移量超过一半，则在另一方向上寻找
        seek_forward = !forward;
        seek_index = quicklist->count - 1 - index; // 计算另一方向上的偏移量
    }

    n = seek_forward ? quicklist->head : quicklist->tail; // 根据方向设置起始节点
    while (likely(n)) { // 循环遍历节点，直到找到目标节点
        if ((accum + n->count) > seek_index) {
            break; // 找到目标节点，退出循环
        } else {
            D("Skipping over (%p) %u at accum %lld", (void *)n, n->count,
              accum); // 跳过当前节点
            accum += n->count; // 累积计算偏移量
            n = seek_forward ? n->next : n->prev; // 移动到下一个节点
        }
    }

    if (!n)
        return NULL; // 如果没有找到目标节点，则返回空迭代器

    /* 修正 accum，使其看起来像是在另一方向上寻找。 */
    if (seek_forward != forward) accum = quicklist->count - n->count - accum; // 修正累积计算的偏移量

    D("Found node: %p at accum %llu, idx %llu, sub+ %llu, sub- %llu", (void *)n,
      accum, index, index - accum, (-index) - 1 + accum);

    quicklistIter *iter = quicklistGetIterator(quicklist, direction); // 获取迭代器
    iter->current = n; // 设置迭代器当前指向的节点
    if (forward) {
        /* forward = normal head-to-tail offset. */
        iter->offset = index - accum; // 设置正向偏移量
    } else {
        /* reverse = need negative offset for tail-to-head, so undo
         * the result of the original index = (-idx) - 1 above. */
        iter->offset = (-index) - 1 + accum; // 设置反向偏移量
    }

    return iter; // 返回迭代器
}

```

### 四、应用
quicklist用于redis的list结构