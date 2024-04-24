# redis中的数据结构
## 字典（Hash Dictionary）
### 一、结构
redis最新unstable源码中，跳表的结构已发生变化（从7.0版本开始）。    
截取源码如下（部分说明写在注释中）：
```c
/* 
 * 该结构已经重dict.h中移动到dict.c中。
 * 每个dictEntry条码结构存储一个键值对，并且通过指针指向下一个同桶内冲突的条目。
 * redis中通过链表的方式解决hash冲突。
 */
struct dictEntry {
    void *key;  // 键
    union {
        void *val;      // 值（通用指针）
        uint64_t u64;   // 64位无符号整数
        int64_t s64;    // 64位有符号整数
        double d;       // 双精度浮点数
    } v;  // 联合体，可存储值的不同类型
    struct dictEntry *next;     /* 同一哈希桶中的下一个条目。*/
};

/* 
 * 该结构仍存在于dict.c中，实际的字典数据结构
 * 相比3.0版本，7.0及之后的版本去除了sizemask,并且取消了dictht这一级结构。
 * 目前dict结构直接使用dictEntry指针数组（dictEntry **ht_table[2]），该数组实际会当做二维数组来存储hash的桶（在后续方法说明中可以查看）。
 * dictEntry是长度为2的数组的原因：在进行重哈希时，redis会启用dictEntry[1]进行数据迁移
 */
struct dict {
    dictType *type;  // 字典的类型特性

    dictEntry **ht_table[2];  // 两个哈希表的数组，用于存储条目指针
    unsigned long ht_used[2];  // 两个哈希表已使用的桶数
    
    long rehashidx; /* 如果 rehashidx == -1，则表示没有进行重新哈希 */

    /* 为了最小化结构填充，将小变量放在最后 */
    int16_t pauserehash; /* 如果 >0，则重新哈希暂停（<0 表示代码执行错误） */
    signed char ht_size_exp[2]; /* 桶大小的指数。（大小 = 1<<exp）*/
    int16_t pauseAutoResize;  /* 如果 >0，则禁止自动调整大小（<0 表示代码执行错误） */
    void *metadata[]; // 元数据数组，用于存储额外的元数据信息
};

/* 
 * 字典类型，用于提供一些多态的回调等
 * 
 */
typedef struct dictType {
    /* 回调函数 */
    uint64_t (*hashFunction)(const void *key);  // 哈希函数
    void *(*keyDup)(dict *d, const void *key);  // 键复制函数
    void *(*valDup)(dict *d, const void *obj);  // 值复制函数
    int (*keyCompare)(dict *d, const void *key1, const void *key2);  // 键比较函数
    void (*keyDestructor)(dict *d, void *key);  // 键析构函数
    void (*valDestructor)(dict *d, void *obj);  // 值析构函数
    int (*resizeAllowed)(size_t moreMem, double usedRatio);  // 是否允许重新调整大小的函数
    /* 在字典初始化/重新哈希开始时调用（旧的和新的哈希表已经创建） */
    void (*rehashingStarted)(dict *d);  // 重新哈希开始时调用的函数
    /* 在所有条目从旧哈希表到新哈希表的初始化/重新哈希完成时调用。此回调完成后，两个哈希表仍然存在，并且在此之后进行清理。 */
    void (*rehashingCompleted)(dict *d);  // 重新哈希完成时调用的函数
    /* 允许字典携带额外的调用者定义的元数据。当分配字典时，额外的内存被初始化为0。*/
    size_t (*dictMetadataBytes)(dict *d);  // 字典元数据字节数的函数

    /* 数据 */
    void *userdata;  // 用户数据

    /* 标志 */
    /* 如果设置了 'no_value' 标志，则表示不使用值，即字典是一个集合。当此标志设置时，无法访问字典条目的值，也无法使用 dictSetKey()。还无法使用条目元数据。 */
    unsigned int no_value:1;  // 无值标志
    /* 如果 no_value = 1 并且所有键都是奇数（LSB = 1），则设置 keys_are_odd = 1 会启用一种优化：存储一个没有分配的字典条目的键。 */
    unsigned int keys_are_odd:1;  // 键为奇数标志
    /* TODO：添加 'keys_are_even' 标志，并在设置了该标志时使用类似的优化。 */
} dictType;

```

### 二、特点
1. 底层采用哈希表实现，使用两个hash表，一个用于存储数据，另一个rehash发生时使用。
2. 字典作为redis数据库的底层实现，在7.0及之后版本，hash算法从murmurhash2,变更为siphash1-2（siphash1-2将单独使用一个小章补充说明）。
3. redis的hash冲突通过链表解决，后进的冲突条目会链到链表头（单向链表，无环）。
4. redis采用渐进式hash,不会一次性完成重hash的操作，进行重hash时，会使用第二个hash表，并且在重hash时，如果条目位置发生变动，会从两个hash表中先后检索。

### 三、rehash相关函数
#### 3.1添加元素函数调用链
```c
#define DICT_OK 0 // 添加成功
#define DICT_ERR 1 // 添加错误
/* 向目标哈希表中添加一个元素 */
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d, key, NULL); // 添加原始元素

    if (!entry) return DICT_ERR;  // 如果没有找到 entry，返回 DICT_ERR
    if (!d->type->no_value) dictSetVal(d, entry, val);  // 如果值类型不为空，设置值
    return DICT_OK;  // 返回成功添加的标志 DICT_OK
}

/* 向字典中原始添加元素，返回新元素的指针，如果已存在则返回现有元素的指针 */
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    /* 获取新键的位置，如果键已存在则返回 NULL */
    void *position = dictFindPositionForInsert(d, key, existing);
    if (!position) return NULL;

    /* 如有必要，复制键 */
    if (d->type->keyDup) key = d->type->keyDup(d, key);

    /* 在指定位置插入键并返回插入后的元素 */
    return dictInsertAtPosition(d, key, position);
}

#define DICTHT_SIZE(exp) ((exp) == -1 ? 0 : (unsigned long)1<<(exp))
#define DICTHT_SIZE_MASK(exp) ((exp) == -1 ? 0 : (DICTHT_SIZE(exp))-1)
/* 查找要插入元素的位置，并返回位置指针 */
void *dictFindPositionForInsert(dict *d, const void *key, dictEntry **existing) {
    unsigned long idx, table;
    dictEntry *he;

    /* 如果 existing 非空，则初始化为 NULL */
    if (existing) *existing = NULL;

    /* 计算键的哈希值 */
    uint64_t hash = dictHashKey(d, key);
    idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[0]);

    /* 如果字典正在进行 rehash */
    if (dictIsRehashing(d)) {
        /* 如果 idx 大于等于 rehashidx 并且 ht0 中存在有效的哈希项 */
        if ((long)idx >= d->rehashidx && d->ht_table[0][idx]) {
            /* 在 ht0 中对 idx 处的桶进行 rehash，以提高 CPU 缓存友好性 */
            _dictBucketRehash(d, idx);
        } else {
            /* 如果哈希项不在 ht0 中，则基于 rehashidx 进行桶的 rehash */
            _dictRehashStep(d);
        }
    }

    /* 如果需要扩展哈希表 */
    _dictExpandIfNeeded(d);

    /* 遍历两个哈希表 */
    for (table = 0; table <= 1; table++) {
        /* 如果是第一个哈希表并且 idx 小于 rehashidx，则跳过 */
        if (table == 0 && (long)idx < d->rehashidx) continue;

        /* 计算哈希值 */
        idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[table]);

        /* 在当前哈希表中查找键是否已存在 */
        he = d->ht_table[table][idx];
        while (he) {
            void *he_key = dictGetKey(he);
            if (key == he_key || dictCompareKeys(d, key, he_key)) {
                /* 如果键已存在，则更新 existing 并返回 NULL */
                if (existing) *existing = he;
                return NULL;
            }
            he = dictGetNext(he);
        }

        /* 如果字典不在 rehash 过程中，则跳出循环 */
        if (!dictIsRehashing(d)) break;
    }

    /* 如果字典正在进行 rehash，则返回第二个哈希表中的桶 */
    dictEntry **bucket = &d->ht_table[dictIsRehashing(d) ? 1 : 0][idx];
    return bucket;
}

/* 在指定位置插入元素到字典中 */
dictEntry *dictInsertAtPosition(dict *d, void *key, void *position) {
    dictEntry **bucket = position; /* 这是一个桶，但 API 隐藏了这一点。 */
    dictEntry *entry;

    /* 如果正在进行 rehash，则插入到第二个哈希表，否则插入到第一个哈希表。
     * 断言所提供的桶是正确的哈希表。 */
    int htidx = dictIsRehashing(d) ? 1 : 0;
    assert(bucket >= &d->ht_table[htidx][0] &&
           bucket <= &d->ht_table[htidx][DICTHT_SIZE_MASK(d->ht_size_exp[htidx])]);

    if (d->type->no_value) {
        if (d->type->keys_are_odd && !*bucket) {
            /* 如果键值对不含值并且键是奇数，并且桶为空，则可以直接存储键到目标桶中。 */
            entry = key;
            assert(entryIsKey(entry)); /* 确保 entry 是键 */
        } else {
            /* 分配一个不含值的条目。 */
            entry = createEntryNoValue(key, *bucket);
        }
    } else {
        /* 分配内存并存储新的条目。
         * 将元素插入到顶部，假设在数据库系统中，最近添加的条目更可能被频繁访问。 */
        entry = zmalloc(sizeof(*entry));
        assert(entryIsNormal(entry)); /* 检查分配的对齐性 */
        entry->key = key;
        entry->next = *bucket;
    }

    *bucket = entry;  // 将 entry 插入到桶中
    d->ht_used[htidx]++;  // 哈希表中的元素数量加一

    return entry;  // 返回插入的元素
}

/* 创建一个不含值的字典条目 */
static inline dictEntry *createEntryNoValue(void *key, dictEntry *next) {
    /* 分配内存给不含值的字典条目 */
    dictEntryNoValue *entry = zmalloc(sizeof(*entry));
    entry->key = key;  // 设置键值
    entry->next = next;  // 设置下一个条目指针
    /* 返回经过特殊处理的字典条目指针，
     * 使用位运算设置最低位表示这是一个不含值的条目 */
    return (dictEntry *)(void *)((uintptr_t)(void *)entry | ENTRY_PTR_NO_VALUE);
}
```

#### 3.2 容量调整函数
在3.1的dictFindPositionForInsert方法中，有一个_dictExpandIfNeeded扩容函数。  
扩容规则参考源码中部分注释的翻译，注意以下扩容和缩容的阈值说明：
```c
/* 使用 dictSetResizeEnabled() 函数可以根据需要禁用哈希表的调整大小和重新哈希操作。这对于 Redis 非常重要，
 * 因为我们使用写时复制，不希望在子进程执行保存操作时移动太多内存。
 *
 * 注意，即使 dict_can_resize 被设置为 DICT_RESIZE_AVOID，仍然无法完全阻止所有调整大小操作：
 *  - 如果元素数量与桶的比例 >= dict_force_resize_ratio，则允许哈希表扩展。
 *  - 如果元素数量与桶的比例 <= 1 / (HASHTABLE_MIN_FILL * dict_force_resize_ratio)，则允许哈希表收缩。 */
static dictResizeEnable dict_can_resize = DICT_RESIZE_ENABLE; /* 哈希表调整大小状态，默认启用 */
static unsigned int dict_force_resize_ratio = 4; /* 哈希表调整大小阈值，默认为 4 */
```

##### 3.2.1 扩容函数
扩容函数源码及注释说明：
```c
/* 如果需要扩展字典大小，则进行扩展操作 */
static void _dictExpandIfNeeded(dict *d) {
    /* 如果字典禁止自动调整大小，直接返回 */
    if (d->pauseAutoResize > 0) return;

    /* 调用字典扩展函数 */
    dictExpandIfNeeded(d);
}

#define DICT_HT_INITIAL_EXP      2
#define DICT_HT_INITIAL_SIZE     (1<<(DICT_HT_INITIAL_EXP))

static dictResizeEnable dict_can_resize = DICT_RESIZE_ENABLE;
static unsigned int dict_force_resize_ratio = 4; // 该安全阈值有变化，最新unstable代码将其调整为4，在7.2及其之前版本还是5

/* 如果需要扩展字典大小，则进行扩展操作 */
int dictExpandIfNeeded(dict *d) {
    /* 如果正在进行增量 rehash，直接返回 */
    if (dictIsRehashing(d)) return DICT_OK;

    /* 如果哈希表为空，则扩展到初始大小 ，此处初始大小为4（1<<2=4） */
    if (DICTHT_SIZE(d->ht_size_exp[0]) == 0) {
        dictExpand(d, DICT_HT_INITIAL_SIZE);
        return DICT_OK;
    }

    /* 如果达到 1:1 的比例，并且可以调整哈希表大小（全局设置），
     * 或者虽然应该避免调整大小但是元素/桶的比率超过“安全”阈值(dict_force_resize_ratio默认为4，即元素数量在超过可承载数量4倍，即便禁止扩容，也要强制扩容)，
     * 则扩展哈希表大小，将桶的数量翻倍 */
    if ((dict_can_resize == DICT_RESIZE_ENABLE &&
         d->ht_used[0] >= DICTHT_SIZE(d->ht_size_exp[0])) ||
        (dict_can_resize != DICT_RESIZE_FORBID &&
         d->ht_used[0] >= dict_force_resize_ratio * DICTHT_SIZE(d->ht_size_exp[0])))
    {
        /* 如果允许调整大小，则进行扩展 */
        if (dictTypeResizeAllowed(d, d->ht_used[0] + 1))
            dictExpand(d, d->ht_used[0] + 1);
        return DICT_OK;
    }
    return DICT_ERR;  // 返回扩展失败的标志 DICT_ERR
}

int dictExpand(dict *d, unsigned long size) {
    return _dictExpand(d, size, NULL);
}

/* 扩展字典的哈希表大小 */
int _dictExpand(dict *d, unsigned long size, int *malloc_failed) {
    /* 如果正在进行 rehash 或者 size 小于哈希表的大小或者小于哈希表已有元素的数量，则大小无效 */
    if (dictIsRehashing(d) || d->ht_used[0] > size || DICTHT_SIZE(d->ht_size_exp[0]) >= size)
        return DICT_ERR;

    /* 调用内部函数进行调整大小操作 */
    return _dictResize(d, size, malloc_failed);
}


/* 调整字典的哈希表大小 */
int _dictResize(dict *d, unsigned long size, int *malloc_failed)
{
    /* 如果 malloc_failed 非空，则初始化为 0 */
    if (malloc_failed) *malloc_failed = 0;

    /* 如果正在进行 rehash，则不允许再次进行 rehash */
    assert(!dictIsRehashing(d));

    /* 新的哈希表 ,_dictNextExp进行扩容的容量计算 */
    dictEntry **new_ht_table;
    unsigned long new_ht_used;
    signed char new_ht_size_exp = _dictNextExp(size);

    /* 检测溢出 */
    size_t newsize = DICTHT_SIZE(new_ht_size_exp);
    if (newsize < size || newsize * sizeof(dictEntry*) < newsize)
        return DICT_ERR;

    /* 如果新的哈希表大小和原大小相同，则不需要进行调整 */
    if (new_ht_size_exp == d->ht_size_exp[0]) return DICT_ERR;

    /* 分配新的哈希表，并将所有指针初始化为 NULL */
    if (malloc_failed) {
        new_ht_table = ztrycalloc(newsize * sizeof(dictEntry*));
        *malloc_failed = new_ht_table == NULL;
        if (*malloc_failed)
            return DICT_ERR;
    } else
        new_ht_table = zcalloc(newsize * sizeof(dictEntry*));

    new_ht_used = 0;

    /* 为增量 rehash 准备第二个哈希表。
     * 即使在第一次初始化时也这样做，以便更方便地触发 rehashingStarted 函数，
     * 我们将在之后立即清理它。 */
    d->ht_size_exp[1] = new_ht_size_exp;
    d->ht_used[1] = new_ht_used;
    d->ht_table[1] = new_ht_table;
    d->rehashidx = 0;

    /* 如果存在 rehashingStarted 函数，则调用 */
    if (d->type->rehashingStarted) d->type->rehashingStarted(d);

    /* 如果是第一次初始化或者第一个哈希表为空，则直接设置第一个哈希表 */
    if (d->ht_table[0] == NULL || d->ht_used[0] == 0) {
        /* 如果存在 rehashingCompleted 函数，则调用 */
        if (d->type->rehashingCompleted) d->type->rehashingCompleted(d);

        /* 释放原有的第一个哈希表，并设置新的哈希表 */
        if (d->ht_table[0]) zfree(d->ht_table[0]);
        d->ht_size_exp[0] = new_ht_size_exp;
        d->ht_used[0] = new_ht_used;
        d->ht_table[0] = new_ht_table;
        _dictReset(d, 1);
        d->rehashidx = -1;
        return DICT_OK;  // 返回调整成功的标志 DICT_OK
    }

    return DICT_OK;
}


/* 使用前导置零法计算下一个哈希表大小的指数值 ,哈希表容量是2的幂 
 * 代码中，8*sizeof(long) 计算出了 long 类型在当前平台下的位数。
 * __builtin_clzl(size-1) 计算了 size-1 的二进制表示中前导零的个数，这个值实际上就是找到了离 size 最近的大于 size 的2的幂次方的值。
*/
static signed char _dictNextExp(unsigned long size)
{
    if (size <= DICT_HT_INITIAL_SIZE) return DICT_HT_INITIAL_EXP; // 如果大小小于等于初始哈希表大小，则返回初始指数
    if (size >= LONG_MAX) return (8*sizeof(long)-1); // 如果大小超出 LONG_MAX，则返回 LONG_MAX 的位数减一

    return 8*sizeof(long) - __builtin_clzl(size-1); // 否则返回 size-1 的前导零位数
}
```
##### 3.2.2 缩容函数
在执行元素删除或者字典解链时，会触发缩容操作。缩容操作的代码如下：
```c
/* 删除一个元素，如果成功则返回 DICT_OK，如果未找到元素则返回 DICT_ERR。 */
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht, key, 0) ? DICT_OK : DICT_ERR;
}

/* 从表中删除一个元素，但不释放键、值和字典条目。如果找到元素（并从表中解除关联），则返回字典条目，
 * 用户应该在以后调用 `dictFreeUnlinkedEntry()` 来释放它。如果未找到键，则返回 NULL。
 *
 * 这个函数在我们想要从哈希表中删除某些内容但想在实际删除条目之前使用其值时非常有用。
 * 如果没有这个函数，该模式将需要两次查找：
 *
 *  entry = dictFind(...);
 *  // 对 entry 进行某些操作
 *  dictDelete(dictionary, entry);
 *
 * 有了这个函数，我们可以避免这样做，而是使用以下方式：
 *
 * entry = dictUnlink(dictionary, entry);
 * // 对 entry 进行某些操作
 * dictFreeUnlinkedEntry(entry); // <- 这不需要再次查找。
 */
dictEntry *dictUnlink(dict *d, const void *key) {
    return dictGenericDelete(d, key, 1);
}

/*
 * 从字典中删除指定键对应的元素。
 * 
 * 参数：
 * - d：字典指针。
 * - key：待删除元素的键。
 * - nofree：是否释放内存的标志，如果为 1 则不释放内存，为 0 则释放内存。
 * 
 * 返回值：
 * - 成功删除元素则返回指向该元素的指针，未找到元素则返回 NULL。
 */
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;         // 哈希值和索引
    dictEntry *he, *prevHe;  // 当前元素和前一个元素
    int table;               // 表号

    // 如果字典为空，则直接返回 NULL
    if (dictSize(d) == 0) return NULL;

    // 计算哈希值和索引
    h = dictHashKey(d, key);
    idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[0]);

    // 如果正在进行 rehash，则进行必要的 rehash 步骤
    if (dictIsRehashing(d)) {
        if ((long)idx >= d->rehashidx && d->ht_table[0][idx]) {
            /* 如果在 ht0 中的 `idx` 处有有效的哈希条目，则在 `idx` 处的桶上进行重新哈希（更加 CPU 缓存友好）。 */
            _dictBucketRehash(d, idx);
        } else {
            /* 如果哈希条目不在 ht0 中，则根据 rehashidx 重新哈希桶（不太 CPU 缓存友好）。 */
            _dictRehashStep(d);
        }
    }
    // 遍历两个哈希表
    for (table = 0; table <= 1; table++) {
        // 如果在进行 rehash 且当前索引小于 rehash 索引，则跳过当前表
        if (table == 0 && (long)idx < d->rehashidx) continue;
        
        // 计算当前表的索引
        idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        he = d->ht_table[table][idx];  // 获取当前元素
        prevHe = NULL;                  // 初始化前一个元素为 NULL

        // 遍历当前桶中的元素
        while(he) {
            void *he_key = dictGetKey(he);  // 获取当前元素的键
            // 如果找到了对应的键，则删除该元素
            if (key == he_key || dictCompareKeys(d, key, he_key)) {
                // 从链表中取消关联元素
                if (prevHe)
                    dictSetNext(prevHe, dictGetNext(he));
                else
                    d->ht_table[table][idx] = dictGetNext(he);
                // 根据 nofree 参数决定是否释放内存
                if (!nofree) {
                    dictFreeUnlinkedEntry(d, he);
                }
                d->ht_used[table]--;    // 更新使用的元素数
                _dictShrinkIfNeeded(d); // 缩小哈希表
                return he;              // 返回被删除的元素指针
            }
            prevHe = he;                // 更新前一个元素指针
            he = dictGetNext(he);       // 移动到下一个元素
        }
        // 如果当前不在进行 rehash，则跳出循环
        if (!dictIsRehashing(d)) break;
    }
    return NULL; // 未找到匹配的元素，返回 NULL
}
/*
 * 你需要在调用 dictUnlink() 后调用该函数来释放元素。即使 'he' 为 NULL 也是安全的。
 */
void dictFreeUnlinkedEntry(dict *d, dictEntry *he) {
    if (he == NULL) return;  // 如果元素为空，则直接返回
    dictFreeKey(d, he);       // 释放键
    dictFreeVal(d, he);       // 释放值
    if (!entryIsKey(he)) zfree(decodeMaskedPtr(he));  // 如果元素不是键值对，则释放元素
}

/*
 * 根据需要缩小哈希表的函数。
 */
static void _dictShrinkIfNeeded(dict *d) 
{
    /* 自动调整大小已禁止，直接返回 */
    if (d->pauseAutoResize > 0) return;

    dictShrinkIfNeeded(d);  // 调用实际缩小哈希表的函数
}

/*
 * 检查是否需要缩小哈希表的函数。
 *
 * 返回值：
 * - DICT_OK 表示成功缩小或字典正在进行 rehash，当前没有其他操作需要执行。
 * - DICT_ERR 表示尚未触发缩小（可以尝试扩展）。
 */
int dictShrinkIfNeeded(dict *d) {
    /* 增量 rehash 已经在进行中，直接返回 */
    if (dictIsRehashing(d)) return DICT_OK;
    
    /* 如果哈希表的大小为初始大小，则不进行缩小 */
    if (DICTHT_SIZE(d->ht_size_exp[0]) <= DICT_HT_INITIAL_SIZE) return DICT_OK;

    /* 如果哈希表的元素数量与桶数的比例低于 1:8，且允许调整哈希表大小或者应该避免调整但比例低于 1:32，
     * 则触发哈希表的缩小。 */
    if ((dict_can_resize == DICT_RESIZE_ENABLE &&
         d->ht_used[0] * HASHTABLE_MIN_FILL <= DICTHT_SIZE(d->ht_size_exp[0])) ||
        (dict_can_resize != DICT_RESIZE_FORBID &&
         d->ht_used[0] * HASHTABLE_MIN_FILL * dict_force_resize_ratio <= DICTHT_SIZE(d->ht_size_exp[0])))
    {
        if (dictTypeResizeAllowed(d, d->ht_used[0]))
            dictShrink(d, d->ht_used[0]);  // 缩小哈希表
        return DICT_OK;
    }
    return DICT_ERR;  // 未触发缩小操作
}
```
#### 3.4 关于dict_can_resize的标识设置
在最新unstable分支代码中，dict_can_resize的修改方法在dict.c中，代码如下
```c
void dictSetResizeEnabled(dictResizeEnable enable) {
    dict_can_resize = enable;
}
```
而调用该方法的函数在server.c中
```c
/* 
 * 此函数在某种后台进程终止时被调用，我们希望在有子进程时避免调整哈希表的大小，
 * 以便与写时复制一起良好地运行（否则当发生调整大小时，会复制许多内存页）。
 * 该函数的目标是根据当前是否有活动的 fork 子进程来更新 dict.c 调整大小或重新哈希表的能力。
 */
void updateDictResizePolicy(void) {
    if (server.in_fork_child != CHILD_TYPE_NONE)
        dictSetResizeEnabled(DICT_RESIZE_FORBID); // 如果在 fork 子进程中，禁止字典调整大小
    else if (hasActiveChildProcess())
        dictSetResizeEnabled(DICT_RESIZE_AVOID); // 如果存在活动的子进程，则避免字典调整大小
    else
        dictSetResizeEnabled(DICT_RESIZE_ENABLE); // 否则允许字典调整大小
}

/* 
 * 如果存在正在进行 RDB 保存、AOF 重写或某个由加载的模块生成的辅助进程，则返回 true。
 */
int hasActiveChildProcess(void) {
    return server.child_pid != -1;
}
```

#### 3.3哈希相关函数调用链
```c
/* 使用字典的哈希函数计算键的哈希值 */
#define dictHashKey(d, key) ((d)->type->hashFunction(key))

/* 根据哈希表大小的指数形式计算实际大小 */
#define DICTHT_SIZE(exp) ((exp) == -1 ? 0 : (unsigned long)1<<(exp))

/* 根据哈希表大小的指数形式计算掩码 */
#define DICTHT_SIZE_MASK(exp) ((exp) == -1 ? 0 : (DICTHT_SIZE(exp))-1)

/* 判断字典是否正在进行 rehash */
#define dictIsRehashing(d) ((d)->rehashidx != -1)

/* 将哈希表中指定位置的桶进行 rehash */
int _dictBucketRehash(dict *d, uint64_t idx) {
    /* 如果暂停 rehash，则直接返回 */
    if (d->pauserehash != 0) return 0;

    /* 计算两个哈希表的大小 */
    unsigned long s0 = DICTHT_SIZE(d->ht_size_exp[0]);
    unsigned long s1 = DICTHT_SIZE(d->ht_size_exp[1]);

    /* 如果禁止调整大小或者没有进行 rehash，则直接返回 */
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;

    /* 如果 dict_can_resize 是 DICT_RESIZE_AVOID，则避免 rehash。
     * - 如果扩展，则阈值为 dict_force_resize_ratio（即 4）。
     * - 如果缩小，则阈值为 1 / (HASHTABLE_MIN_FILL * dict_force_resize_ratio)（即 1/32）。 */
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 < dict_force_resize_ratio * s0) ||
         (s1 < s0 && s0 < HASHTABLE_MIN_FILL * dict_force_resize_ratio * s1)))
    {
        return 0;
    }

    /* 对指定位置的桶进行 rehash */
    rehashEntriesInBucketAtIndex(d, idx);
    /* 检查 rehash 是否完成 */
    dictCheckRehashingCompleted(d);
    return 1;
}

/* 对指定位置的桶中的元素进行 rehash */
static void rehashEntriesInBucketAtIndex(dict *d, uint64_t idx) {
    dictEntry *de = d->ht_table[0][idx];
    uint64_t h;
    dictEntry *nextde;
    while (de) {
        nextde = dictGetNext(de);
        void *key = dictGetKey(de);
        /* 在新哈希表中获取索引 */
        if (d->ht_size_exp[1] > d->ht_size_exp[0]) {
            h = dictHashKey(d, key) & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
        } else {
            /* 缩小表格。 表格大小是 2 的幂次方，
             * 因此我们可以简单地将较大表中的桶索引掩码为较小表中的桶索引。 */
            h = idx & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
        }
        if (d->type->no_value) {
            if (d->type->keys_are_odd && !d->ht_table[1][h]) {
                /* 目标桶为空且可以直接存储键，无需分配条目。释放旧条目（如果有）。 */
                assert(entryIsKey(key));
                if (!entryIsKey(de)) zfree(decodeMaskedPtr(de));
                de = key;
            } else if (entryIsKey(de)) {
                /* 没有分配的条目但是需要一个。 */
                de = createEntryNoValue(key, d->ht_table[1][h]);
            } else {
                /* 将现有条目移动到目标哈希表并更新 'next' 字段。 */
                assert(entryIsNoValue(de));
                dictSetNext(de, d->ht_table[1][h]);
            }
        } else {
            dictSetNext(de, d->ht_table[1][h]);
        }
        d->ht_table[1][h] = de;
        d->ht_used[0]--;
        d->ht_used[1]++;
        de = nextde;
    }
    d->ht_table[0][idx] = NULL;
}

/* 检查 rehash 是否完成 */
static int dictCheckRehashingCompleted(dict *d) {
    /* 如果第一个哈希表不为空，则返回 0 */
    if (d->ht_used[0] != 0) return 0;
    
    /* 如果存在 rehashingCompleted 函数，则调用 */
    if (d->type->rehashingCompleted) d->type->rehashingCompleted(d);
    zfree(d->ht_table[0]);
    /* 将新哈希表复制到旧哈希表 */
    d->ht_table[0] = d->ht_table[1];
    d->ht_used[0] = d->ht_used[1];
    d->ht_size_exp[0] = d->ht_size_exp[1];
    _dictReset(d, 1);
    d->rehashidx = -1;
    return 1;
}


/* 执行字典的单步 rehash */
static void _dictRehashStep(dict *d) {
    /* 如果未暂停 rehash，则执行 1 步 rehash */
    if (d->pauserehash == 0) dictRehash(d, 1);
}

/* 执行字典的 rehash */
int dictRehash(dict *d, int n) {
    /* 最大访问空桶次数 */
    int empty_visits = n * 10;
    /* 获取两个哈希表的大小 */
    unsigned long s0 = DICTHT_SIZE(d->ht_size_exp[0]);
    unsigned long s1 = DICTHT_SIZE(d->ht_size_exp[1]);
    
    /* 如果禁止调整大小或者没有进行 rehash，则直接返回 */
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    
    /* 如果 dict_can_resize 是 DICT_RESIZE_AVOID，则避免 rehash。
     * - 如果扩展，则阈值为 dict_force_resize_ratio（即 4）。
     * - 如果缩小，则阈值为 1 / (HASHTABLE_MIN_FILL * dict_force_resize_ratio)（即 1/32）。 */
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 < dict_force_resize_ratio * s0) ||
         (s1 < s0 && s0 < HASHTABLE_MIN_FILL * dict_force_resize_ratio * s1)))
    {
        return 0;
    }

    /* 进行指定步数的 rehash */
    while (n-- && d->ht_used[0] != 0) {
        /* 注意，rehashidx 不会溢出，因为我们确信还有更多元素，因为 ht[0].used != 0 */
        assert(DICTHT_SIZE(d->ht_size_exp[0]) > (unsigned long)d->rehashidx);
        while (d->ht_table[0][d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        /* 将此桶中的所有键从旧哈希表移动到新哈希表 */
        rehashEntriesInBucketAtIndex(d, d->rehashidx);
        d->rehashidx++;
    }

    /* 检查 rehash 是否完成 */
    return !dictCheckRehashingCompleted(d);
}
```

#### 3.4默认hash函数
```c
/* -------------------------- private prototypes ---------------------------- */

static void _dictExpandIfNeeded(dict *d);
static void _dictShrinkIfNeeded(dict *d);
static signed char _dictNextExp(unsigned long size);
static int _dictInit(dict *d, dictType *type);
static dictEntry *dictGetNext(const dictEntry *de);
static dictEntry **dictGetNextRef(dictEntry *de);
static void dictSetNext(dictEntry *de, dictEntry *next);

/* -------------------------- hash functions -------------------------------- */

static uint8_t dict_hash_function_seed[16];

/* 设置哈希函数种子 */
void dictSetHashFunctionSeed(uint8_t *seed) {
    memcpy(dict_hash_function_seed, seed, sizeof(dict_hash_function_seed));
}

/* 获取哈希函数种子 */
uint8_t *dictGetHashFunctionSeed(void) {
    return dict_hash_function_seed;
}

/* 默认哈希函数使用 siphash.c 中的 SipHash 实现 */

/* SipHash 哈希函数 */
uint64_t siphash(const uint8_t *in, const size_t inlen, const uint8_t *k);
/* SipHash 不区分大小写的哈希函数 */
uint64_t siphash_nocase(const uint8_t *in, const size_t inlen, const uint8_t *k);

/* 生成哈希函数 */
uint64_t dictGenHashFunction(const void *key, size_t len) {
    return siphash(key, len, dict_hash_function_seed);
}

/* 生成不区分大小写的哈希函数 */
uint64_t dictGenCaseHashFunction(const unsigned char *buf, size_t len) {
    return siphash_nocase(buf, len, dict_hash_function_seed);
}
```

#### 3.5查询和查找key部分方法

```c
/* 在字典中查找指定键对应的哈希表节点 */
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he; /* 哈希表节点 */
    uint64_t h, idx, table; /* 哈希值、索引、哈希表序号 */

    /* 如果字典为空，则返回空指针 */
    if (dictSize(d) == 0) return NULL;

    /* 计算键的哈希值 */
    h = dictHashKey(d, key);
    idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[0]); /* 计算哈希值对应的索引 */

    /* 如果正在进行 rehash */
    if (dictIsRehashing(d)) {
        if ((long)idx >= d->rehashidx && d->ht_table[0][idx]) {
            /* 如果 ht0 中在索引 idx 处有有效的哈希节点，则在该索引处执行 rehash */
            _dictBucketRehash(d, idx);
        } else {
            /* 如果哈希节点不在 ht0 中，则基于 rehashidx 对桶进行 rehash */
            _dictRehashStep(d);
        }
    }

    /* 遍历两个哈希表，查找指定键对应的哈希节点 */
    for (table = 0; table <= 1; table++) {
        if (table == 0 && (long)idx < d->rehashidx) continue;
        idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        he = d->ht_table[table][idx];
        while (he) {
            void *he_key = dictGetKey(he);
            /* 比较键是否相等，如果相等则返回当前哈希节点 */
            if (key == he_key || dictCompareKeys(d, key, he_key))
                return he;
            he = dictGetNext(he);
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL; /* 未找到对应的哈希节点 */
}

/* 从字典中获取一些键的哈希表节点 */
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count) {
    unsigned long j; /* 内部哈希表 ID，0 或 1 */
    unsigned long tables; /* 1 或 2 个哈希表 */
    unsigned long stored = 0, maxsizemask;
    unsigned long maxsteps;

    /* 如果字典大小小于指定数量，则将 count 设置为字典大小 */
    if (dictSize(d) < count) count = dictSize(d);
    maxsteps = count * 10;

    /* 尝试进行与 'count' 成比例的 rehash 工作 */
    for (j = 0; j < count; j++) {
        if (dictIsRehashing(d))
            _dictRehashStep(d);
        else
            break;
    }

    /* 确定当前使用的哈希表数量 */
    tables = dictIsRehashing(d) ? 2 : 1;
    maxsizemask = DICTHT_SIZE_MASK(d->ht_size_exp[0]);
    if (tables > 1 && maxsizemask < DICTHT_SIZE_MASK(d->ht_size_exp[1]))
        maxsizemask = DICTHT_SIZE_MASK(d->ht_size_exp[1]);

    /* 从较大的哈希表中选择一个随机点 */
    unsigned long i = randomULong() & maxsizemask;
    unsigned long emptylen = 0; /* 连续空桶数量 */
    while (stored < count && maxsteps--) {
        for (j = 0; j < tables; j++) {
            /* dict.c 中 rehashing 的不变性：在 rehash 过程中已经访问的索引
             * 在 ht[0] 中没有填充的桶，因此我们可以跳过在 rehashing 过程中
             * 0 到 idx-1 之间的索引的 ht[0]。 */
            if (tables == 2 && j == 0 && i < (unsigned long)d->rehashidx) {
                /* 此外，如果当前超出第二个哈希表的范围，则在当前重哈希索引
                 * 之后的两个哈希表中都没有元素，因此如果可能，我们就跳过这一部分。
                 * （当从大到小的哈希表切换时会发生这种情况）。 */
                if (i >= DICTHT_SIZE(d->ht_size_exp[1]))
                    i = d->rehashidx;
                else
                    continue;
            }
            if (i >= DICTHT_SIZE(d->ht_size_exp[j])) continue; /* 超出此哈希表的范围 */
            dictEntry *he = d->ht_table[j][i];

            /* 计算连续空桶数量，并在达到 'count'（最小为 5）时跳到其他位置。 */
            if (he == NULL) {
                emptylen++;
                if (emptylen >= 5 && emptylen > count) {
                    i = randomULong() & maxsizemask;
                    emptylen = 0;
                }
            } else {
                emptylen = 0;
                while (he) {
                    /* 收集所有非空桶中的元素。
                     * 为了避免无法对长链的末尾进行采样的问题，
                     * 我们利用了水库抽样算法来优化采样过程。这意味着即使达到了最大采样数，
                     * 我们仍然会继续采样直到到达链的末尾。详情请参阅 https://en.wikipedia.org/wiki/Reservoir_sampling。 */
                    if (stored < count) {
                        des[stored] = he;
                    } else {
                        unsigned long r = randomULong() % (stored + 1);
                        if (r < count) des[r] = he;
                    }

                    he = dictGetNext(he);
                    stored++;
                }
                if (stored >= count) goto end;
            }
        }
        i = (i + 1) & maxsizemask;
    }

end:
    return stored > count ? count : stored;
}
```

### 四、应用
redis数据库和哈希键

### 五、其他
1.在《redis设计与实现》一书中，有介绍关于字典的容量调整，会判断服务器目前是否在执行BGSAVE或者BGREWRITEAOF，但在字典的实现源码文件中并未找到，若在其他部分找到相关逻辑，会进行补充。
2.在《redis设计与实现》一书中，缩容的操作发生在负载因子小于0.1的情况下，但从最新unstable源码分析，已经修改为元素/桶数<8分之1时。
3.在《redis设计与实现》一书中，扩容操作发生在“未执行BGSAVE或者BGREWRITEAOF，且负载因子大于1”或“执行BGSAVE或者BGREWRITEAOF，且复杂因子大于5”这两种情况下，但通过分析最新unstable源码，现有逻辑已经变化，可查看3.2.1中源码的扩容判断部分的注释。
TODO 补充时序图或者扩容操作图。