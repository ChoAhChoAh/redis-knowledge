# redis中的数据结构
## 整形数组
### 一、结构
整形数组使用在redis的set中，当redis的set键均为整数，且数量不多时，底层使用该数据结构实现。
该数据结构的定位如下：
```c
typedef struct intset {
    uint32_t encoding; // 编码方式
    uint32_t length; // 元素数量
    int8_t contents[];  // 保存元素的数组
} intset;
```
虽然上述代码中，元素数组的类型为int8_t，但实际在intset.c中，数据保存的格式会根据encoding来调整。  
```c
static uint8_t _intsetValueEncoding(int64_t v) {
    if (v < INT32_MIN || v > INT32_MAX) //小于-2147483648 或 大于 2147483647 ，使用int64
        return INTSET_ENC_INT64;
    else if (v < INT16_MIN || v > INT16_MAX) //小于-32768 或 大于 32767 ，使用int32
        return INTSET_ENC_INT32;
    else  // 其他情况则使用int16
        return INTSET_ENC_INT16;
}
```
补充：  
int8的范围 -128 ~ 127  
int64的范围 -9223372036854775808 ~ 9223372036854775807

### 二、特性
1. 当redis的set仅存储少量整形数据时，底层使用intset。
2. intset底层为数组，连续的存储空间，提高内存利用率，减少内存碎片。
3. 升级：在上文的结构介绍中，元素数组是int8_t类型，但实际存储值得类型可为int16_t、int32_t、int64_t,根据encoding的编码类型而定。  
初始为int16_t,但如果存储的新元素的整数范围大于16比特（2^8：-32768~32767），就会根据实际的数据长度，升级为int32_t或者int64_t，这么做的优势在于可以节省内存空间，只在必要时扩容；提升类型灵活性，不会出现类型错误。
4. 降级：不支持容量编码类型的降级操作。

### 三、部分函数源码
#### 3.1 初始化
```c
/* 创建一个空的整数集合 */
intset *intsetNew(void) {
intset *is = zmalloc(sizeof(intset)); // 分配内存空间以存储整数集合
is->encoding = intrev32ifbe(INTSET_ENC_INT16); // 将编码方式设置为 INT16，并根据系统字节序进行转换
is->length = 0; // 设置集合的长度为 0
return is; // 返回创建的整数集合
}
```
上述代码中，interv32ifbe用于转换大小端机器编码顺序统一，该方法在endianconv.h的宏定义中。实际被调用的方法intrev32在endianconv.c代码中。
```c
/* 宏定义 */
/* 这是对函数的变体，仅在目标主机为大端序时执行实际的转换 */
#if (BYTE_ORDER == LITTLE_ENDIAN)
// 如果目标主机为小端序，则不执行转换，仅返回原始值
#define memrev16ifbe(p) ((void)(0)) // 将宏 memrev16ifbe 定义为空操作
#define memrev32ifbe(p) ((void)(0)) // 将宏 memrev32ifbe 定义为空操作
#define memrev64ifbe(p) ((void)(0)) // 将宏 memrev64ifbe 定义为空操作
#define intrev16ifbe(v) (v) // 将宏 intrev16ifbe 定义为返回原始值
#define intrev32ifbe(v) (v) // 将宏 intrev32ifbe 定义为返回原始值
#define intrev64ifbe(v) (v) // 将宏 intrev64ifbe 定义为返回原始值
#else
// 如果目标主机为大端序，则执行实际的转换
#define memrev16ifbe(p) memrev16(p) // 将宏 memrev16ifbe 定义为调用函数 memrev16
#define memrev32ifbe(p) memrev32(p) // 将宏 memrev32ifbe 定义为调用函数 memrev32
#define memrev64ifbe(p) memrev64(p) // 将宏 memrev64ifbe 定义为调用函数 memrev64
#define intrev16ifbe(v) intrev16(v) // 将宏 intrev16ifbe 定义为调用函数 intrev16
#define intrev32ifbe(v) intrev32(v) // 将宏 intrev32ifbe 定义为调用函数 intrev32
#define intrev64ifbe(v) intrev64(v) // 将宏 intrev64ifbe 定义为调用函数 intrev64
#endif

/* 举例 */
void memrev32(void *p) {
    unsigned char *x = p, t;

    t = x[0];
    x[0] = x[3];
    x[3] = t;
    t = x[1];
    x[1] = x[2];
    x[2] = t;
}
```
上述的memrev32方法，是将指针 *p 指向的32位无符号整数从小端序（little endian）转换为大端序（big endian）。  
在小端序中，低字节存储在内存的低地址处，而在大端序中，高字节存储在低地址处。代码中使用了一个指针 *p，它被转换为一个指向 unsigned char 类型的指针 x。  
这是因为希望以字节的方式处理整数，以便可以直接访问每个字节。  
代码通过交换每个字节的位置来完成小端序到大端序的转换。  
具体流程：将指针 x 指向的内存中的第一个字节（x[0]）与第四个字节（x[3]）进行交换，然后将第二个字节（x[1]）与第三个字节（x[2]）进行交换。经过这个函数处理后，原来小端序存储的整数就会被转换成大端序存储。

#### 3.2 添加元素

```c
/* 向 intset 中插入一个整数 */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value); // 确定整数的编码方式
    uint32_t pos; // 插入位置
    if (success) *success = 1; // 初始化 success 为成功状态

    /* 如果需要的话，升级编码方式。如果我们需要升级，我们知道该值应该要么追加（如果 > 0）要么预置（如果 < 0），
     * 因为它位于现有值的范围之外。 */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* 这总是成功的，所以我们不需要传递 *success。 */
        return intsetUpgradeAndAdd(is,value); // 升级 intset 的编码方式，并添加值
    } else {
        /* 如果值已经存在于集合中，则中止。
         * 当无法找到值时，此调用将 "pos" 填充为插入值的正确位置。 */
        if (intsetSearch(is,value,&pos)) { // 搜索值在集合中的位置
            if (success) *success = 0; // 将 success 设置为失败状态
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1); // 调整 intset 的大小以容纳新值
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1); // 移动尾部以腾出插入位置
    }

    _intsetSet(is,pos,value); // 在指定位置设置新值
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1); // 更新 intset 的长度
    return is;
}

/* 将 intset 升级到更大的编码方式，并插入给定的整数 */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding); // 获取当前编码方式
    uint8_t newenc = _intsetValueEncoding(value); // 获取给定整数的编码方式
    int length = intrev32ifbe(is->length); // 获取 intset 的长度
    int prepend = value < 0 ? 1 : 0; // 标记是否在前方插入

    /* 首先设置新的编码方式并调整大小 */
    is->encoding = intrev32ifbe(newenc); // 将 intset 编码方式设置为新编码方式
    is = intsetResize(is,intrev32ifbe(is->length)+1); // 调整 intset 的大小以容纳新值

    /* 从后往前升级，以避免覆盖值。
     * 注意，“prepend” 变量用于确保在 intset 的开头或末尾有一个空位。 */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc)); // 将原来的值逐个设置到新的位置

    /* 设置值在开头或末尾。 */
    if (prepend)
        _intsetSet(is,0,value); // 如果在前方插入，则将值设置在开头
    else
        _intsetSet(is,intrev32ifbe(is->length),value); // 如果在后方插入，则将值设置在末尾
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1); // 更新 intset 的长度
    return is; // 返回更新后的 intset
}

/* 调整 intset 的大小 */
static intset *intsetResize(intset *is, uint32_t len) {
    uint64_t size = (uint64_t)len*intrev32ifbe(is->encoding); // 计算需要的新大小
    assert(size <= SIZE_MAX - sizeof(intset)); // 断言新大小不超过 SIZE_MAX 减去 intset 的大小

    is = zrealloc(is,sizeof(intset)+size); // 重新分配内存以适应新的大小
    return is; // 返回调整大小后的 intset
}

/* 使用配置的编码方式设置指定位置的值 */
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding); // 获取编码方式

    if (encoding == INTSET_ENC_INT64) { // 如果编码方式为 INT64
        ((int64_t*)is->contents)[pos] = value; // 将值设置到指定位置
        memrev64ifbe(((int64_t*)is->contents)+pos); // 根据目标主机的字节序调整内存中的字节顺序
    } else if (encoding == INTSET_ENC_INT32) { // 如果编码方式为 INT32
        ((int32_t*)is->contents)[pos] = value; // 将值设置到指定位置
        memrev32ifbe(((int32_t*)is->contents)+pos); // 根据目标主机的字节序调整内存中的字节顺序
    } else { // 如果编码方式为 INT16
        ((int16_t*)is->contents)[pos] = value; // 将值设置到指定位置
        memrev16ifbe(((int16_t*)is->contents)+pos); // 根据目标主机的字节序调整内存中的字节顺序
    }
}
```
#### 3.3 查询元素

```c
/* 确定一个值是否属于该集合 */
uint8_t intsetFind(intset *is, int64_t value) {
    uint8_t valenc = _intsetValueEncoding(value); // 确定值的编码方式
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL); // 返回值是否在集合中
}



/* 在 intset 中搜索 "value" 的位置。当找到值时返回 1，并将 "pos" 设置为值在 intset 中的位置。
 * 当值不在 intset 中时返回 0，并将 "pos" 设置为可以插入 "value" 的位置。 */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1; // 定义搜索范围的最小值、最大值和中间值
    int64_t cur = -1; // 当前值初始化为 -1

    /* 当集合为空时，永远找不到值 */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0; // 设置 pos 为 0
        return 0; // 返回 0，表示值不在集合中
    } else {
        /* 检查我们无法找到值但知道插入位置的情况。 */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length); // 设置 pos 为最大位置
            return 0; // 返回 0，表示值不在集合中
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0; // 设置 pos 为 0
            return 0; // 返回 0，表示值不在集合中
        }
    }

    while(max >= min) { // 二分搜索
        mid = ((unsigned int)min + (unsigned int)max) >> 1; // 计算中间位置
        cur = _intsetGet(is,mid); // 获取中间位置的值
        if (value > cur) {
            min = mid+1; // 更新最小值
        } else if (value < cur) {
            max = mid-1; // 更新最大值
        } else {
            break; // 找到值，退出循环
        }
    }

    if (value == cur) { // 如果找到值
        if (pos) *pos = mid; // 设置 pos 为找到的位置
        return 1; // 返回 1，表示值在集合中
    } else { // 如果值不在集合中
        if (pos) *pos = min; // 设置 pos 为可以插入值的位置
        return 0; // 返回 0，表示值不在集合中
    }
}


/* 返回指定位置的值，使用配置的编码方式。 */
static int64_t _intsetGet(intset *is, int pos) {
    return _intsetGetEncoded(is,pos,intrev32ifbe(is->encoding)); // 调用 _intsetGetEncoded 函数获取指定位置的值
}


/* 根据给定的编码方式返回指定位置的值。 */
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64; // 用于存储 INT64 类型的值
    int32_t v32; // 用于存储 INT32 类型的值
    int16_t v16; // 用于存储 INT16 类型的值

    if (enc == INTSET_ENC_INT64) { // 如果编码方式为 INT64
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64)); // 从内容中复制 INT64 类型的值
        memrev64ifbe(&v64); // 根据目标主机的字节序调整值的字节顺序
        return v64; // 返回 INT64 类型的值
    } else if (enc == INTSET_ENC_INT32) { // 如果编码方式为 INT32
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32)); // 从内容中复制 INT32 类型的值
        memrev32ifbe(&v32); // 根据目标主机的字节序调整值的字节顺序
        return v32; // 返回 INT32 类型的值
    } else { // 如果编码方式为 INT16
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16)); // 从内容中复制 INT16 类型的值
        memrev16ifbe(&v16); // 根据目标主机的字节序调整值的字节顺序
        return v16; // 返回 INT16 类型的值
    }
}
```

#### 3.4 删除元素

```c
/* 从 intset 中删除整数 */
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value); // 确定整数的编码方式
    uint32_t pos; // 用于存储整数在 intset 中的位置
    if (success) *success = 0; // 初始化 success 为失败状态

    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) { // 如果整数在 intset 中且找到了位置
        uint32_t len = intrev32ifbe(is->length); // 获取 intset 的长度

        /* 我们知道可以删除 */
        if (success) *success = 1; // 设置 success 为成功状态

        /* 用尾部覆盖值并更新长度 */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos); // 将尾部的值覆盖到被删除的位置
        is = intsetResize(is,len-1); // 调整 intset 大小以删除值
        is->length = intrev32ifbe(len-1); // 更新 intset 的长度
    }
    return is; // 返回删除操作后的 intset
}


/* 将 intset 中的值移动到新的位置 */
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst; // 定义源地址和目标地址的指针
    uint32_t bytes = intrev32ifbe(is->length)-from; // 计算需要移动的字节数
    uint32_t encoding = intrev32ifbe(is->encoding); // 获取 intset 的编码方式

    if (encoding == INTSET_ENC_INT64) { // 如果编码方式为 INT64
        src = (int64_t*)is->contents+from; // 计算源地址
        dst = (int64_t*)is->contents+to; // 计算目标地址
        bytes *= sizeof(int64_t); // 计算需要移动的字节数
    } else if (encoding == INTSET_ENC_INT32) { // 如果编码方式为 INT32
        src = (int32_t*)is->contents+from; // 计算源地址
        dst = (int32_t*)is->contents+to; // 计算目标地址
        bytes *= sizeof(int32_t); // 计算需要移动的字节数
    } else { // 如果编码方式为 INT16
        src = (int16_t*)is->contents+from; // 计算源地址
        dst = (int16_t*)is->contents+to; // 计算目标地址
        bytes *= sizeof(int16_t); // 计算需要移动的字节数
    }
    memmove(dst,src,bytes); // 使用 memmove 函数将值移动到新的位置
}

```

### 四、应用
Set类型只存储少量整形时会使用intset。