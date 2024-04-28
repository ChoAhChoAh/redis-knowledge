# redis中的数据结构
## redis对象（redisObject）
### 一、结构
redisObject的结构如下代码,redis采用的变量后使用冒号和数值的定义方法，是C语言中的位域定义方法，可以用来有效地节省内存开销。  
解释来说：当一个变量占用不了一个数据类型的所有 bits 时，就可以使用位域定义方法，把一个数据类型中的 bits，划分成多个位域，每个位域占一定的 bit 数。这样一来，一个数据类型的所有 bits 就可以定义多个变量了，从而也就有效节省了内存开销。  
下面redisObject的定义中type、encoding和lru三个变量，它们的数据类型都是 unsigned。已知一个unsigned类型是 4 字节，但这三个变量，是分别占用了一个unsigned类型4字节中的 4bits、4bits和24bits。因此，相较于三个变量，每个变量用一个4字节的 unsigned 类型定义来说，使用位域定义方法可以让三个变量只用4字节，最后就能节省8字节的开销。
```c
struct redisObject {
    unsigned type:4;// redisObject的数据类型，4个bits
    unsigned encoding:4; //redisObject的编码类型，4个bits
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount; //redisObject的引用计数，4个字节
    void *ptr; //指向值的指针，8个字节
};
```
上面代码中，type为对外类型，encoding为对内类型。
type定义的类型如下，即常用的五种数据类型：
```c
/* 实际的redis对象 */
#define OBJ_STRING 0    /* 字符串对象. */
#define OBJ_LIST 1      /* 列表对象. */
#define OBJ_SET 2       /* 集合对象. */
#define OBJ_ZSET 3      /* 有序集合对象. */
#define OBJ_HASH 4      /* 哈希对象. */
```

encoding定义的类型为redis内部实现上述五种类型的实际使用数据结构，定义如下：
```c
/* 对象编码。某些类型的对象，如字符串和哈希表，可以以多种方式进行内部表示。
 * 对象的 'encoding' 字段被设置为以下字段之一，表示此对象的编码方式。 */
#define OBJ_ENCODING_RAW 0             /* 原始表示方式 */
#define OBJ_ENCODING_INT 1             /* 编码为整数 */
#define OBJ_ENCODING_HT 2              /* 编码为哈希表 */
#define OBJ_ENCODING_ZIPMAP 3          /* 不再使用：旧的哈希编码方式 */
#define OBJ_ENCODING_LINKEDLIST 4      /* 不再使用：旧的列表编码方式 */
#define OBJ_ENCODING_ZIPLIST 5         /* 不再使用：旧的列表/哈希/有序集编码方式 */
#define OBJ_ENCODING_INTSET 6          /* 编码为整数集合 */
#define OBJ_ENCODING_SKIPLIST 7        /* 编码为跳跃表 */
#define OBJ_ENCODING_EMBSTR 8          /* 嵌入式 sds 字符串编码方式 */
#define OBJ_ENCODING_QUICKLIST 9       /* 编码为 listpack 的链表 */
#define OBJ_ENCODING_STREAM 10         /* 编码为 listpack 的基数树 */
#define OBJ_ENCODING_LISTPACK 11       /* 编码为 listpack */
```

### 二、特性
1. 为多种数据类型提供统一的表示方式
2. 同一种数据类型，底层可以对应不同实现，节省内存
3. 支持对象共享和引用计数，共享对象存储一份，可多次使用，节省内存
#### 2.1 创建共享对象
redis为了节省内存，在服务启动时会创建很多共享对象，这些共享对象会被重复引用，以减少冗余存储对于内存的浪费。
```c
void createSharedObjects(void) {
    int j;

    /* Shared command responses */
    shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
    shared.emptybulk = createObject(OBJ_STRING,sdsnew("$0\r\n\r\n"));
    shared.czero = createObject(OBJ_STRING,sdsnew(":0\r\n"));
    shared.cone = createObject(OBJ_STRING,sdsnew(":1\r\n"));
    shared.emptyarray = createObject(OBJ_STRING,sdsnew("*0\r\n"));
    shared.pong = createObject(OBJ_STRING,sdsnew("+PONG\r\n"));
    shared.queued = createObject(OBJ_STRING,sdsnew("+QUEUED\r\n"));
    shared.emptyscan = createObject(OBJ_STRING,sdsnew("*2\r\n$1\r\n0\r\n*0\r\n"));
    shared.space = createObject(OBJ_STRING,sdsnew(" "));
    shared.plus = createObject(OBJ_STRING,sdsnew("+"));

    /* Shared command error responses */
    shared.wrongtypeerr = createObject(OBJ_STRING,sdsnew(
        "-WRONGTYPE Operation against a key holding the wrong kind of value\r\n"));
    shared.err = createObject(OBJ_STRING,sdsnew("-ERR\r\n"));
    shared.nokeyerr = createObject(OBJ_STRING,sdsnew(
        "-ERR no such key\r\n"));
    shared.syntaxerr = createObject(OBJ_STRING,sdsnew(
        "-ERR syntax error\r\n"));
    shared.sameobjecterr = createObject(OBJ_STRING,sdsnew(
        "-ERR source and destination objects are the same\r\n"));
    shared.outofrangeerr = createObject(OBJ_STRING,sdsnew(
        "-ERR index out of range\r\n"));
    shared.noscripterr = createObject(OBJ_STRING,sdsnew(
        "-NOSCRIPT No matching script. Please use EVAL.\r\n"));
    shared.loadingerr = createObject(OBJ_STRING,sdsnew(
        "-LOADING Redis is loading the dataset in memory\r\n"));
    shared.slowevalerr = createObject(OBJ_STRING,sdsnew(
        "-BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.\r\n"));
    shared.slowscripterr = createObject(OBJ_STRING,sdsnew(
        "-BUSY Redis is busy running a script. You can only call FUNCTION KILL or SHUTDOWN NOSAVE.\r\n"));
    shared.slowmoduleerr = createObject(OBJ_STRING,sdsnew(
        "-BUSY Redis is busy running a module command.\r\n"));
    shared.masterdownerr = createObject(OBJ_STRING,sdsnew(
        "-MASTERDOWN Link with MASTER is down and replica-serve-stale-data is set to 'no'.\r\n"));
    shared.bgsaveerr = createObject(OBJ_STRING,sdsnew(
        "-MISCONF Redis is configured to save RDB snapshots, but it's currently unable to persist to disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n"));
    shared.roslaveerr = createObject(OBJ_STRING,sdsnew(
        "-READONLY You can't write against a read only replica.\r\n"));
    shared.noautherr = createObject(OBJ_STRING,sdsnew(
        "-NOAUTH Authentication required.\r\n"));
    shared.oomerr = createObject(OBJ_STRING,sdsnew(
        "-OOM command not allowed when used memory > 'maxmemory'.\r\n"));
    shared.execaborterr = createObject(OBJ_STRING,sdsnew(
        "-EXECABORT Transaction discarded because of previous errors.\r\n"));
    shared.noreplicaserr = createObject(OBJ_STRING,sdsnew(
        "-NOREPLICAS Not enough good replicas to write.\r\n"));
    shared.busykeyerr = createObject(OBJ_STRING,sdsnew(
        "-BUSYKEY Target key name already exists.\r\n"));

    /* The shared NULL depends on the protocol version. */
    shared.null[0] = NULL;
    shared.null[1] = NULL;
    shared.null[2] = createObject(OBJ_STRING,sdsnew("$-1\r\n"));
    shared.null[3] = createObject(OBJ_STRING,sdsnew("_\r\n"));

    shared.nullarray[0] = NULL;
    shared.nullarray[1] = NULL;
    shared.nullarray[2] = createObject(OBJ_STRING,sdsnew("*-1\r\n"));
    shared.nullarray[3] = createObject(OBJ_STRING,sdsnew("_\r\n"));

    shared.emptymap[0] = NULL;
    shared.emptymap[1] = NULL;
    shared.emptymap[2] = createObject(OBJ_STRING,sdsnew("*0\r\n"));
    shared.emptymap[3] = createObject(OBJ_STRING,sdsnew("%0\r\n"));

    shared.emptyset[0] = NULL;
    shared.emptyset[1] = NULL;
    shared.emptyset[2] = createObject(OBJ_STRING,sdsnew("*0\r\n"));
    shared.emptyset[3] = createObject(OBJ_STRING,sdsnew("~0\r\n"));

    for (j = 0; j < PROTO_SHARED_SELECT_CMDS; j++) {
        char dictid_str[64];
        int dictid_len;

        dictid_len = ll2string(dictid_str,sizeof(dictid_str),j);
        shared.select[j] = createObject(OBJ_STRING,
            sdscatprintf(sdsempty(),
                "*2\r\n$6\r\nSELECT\r\n$%d\r\n%s\r\n",
                dictid_len, dictid_str));
    }
    shared.messagebulk = createStringObject("$7\r\nmessage\r\n",13);
    shared.pmessagebulk = createStringObject("$8\r\npmessage\r\n",14);
    shared.subscribebulk = createStringObject("$9\r\nsubscribe\r\n",15);
    shared.unsubscribebulk = createStringObject("$11\r\nunsubscribe\r\n",18);
    shared.ssubscribebulk = createStringObject("$10\r\nssubscribe\r\n", 17);
    shared.sunsubscribebulk = createStringObject("$12\r\nsunsubscribe\r\n", 19);
    shared.smessagebulk = createStringObject("$8\r\nsmessage\r\n", 14);
    shared.psubscribebulk = createStringObject("$10\r\npsubscribe\r\n",17);
    shared.punsubscribebulk = createStringObject("$12\r\npunsubscribe\r\n",19);

    /* Shared command names */
    shared.del = createStringObject("DEL",3);
    shared.unlink = createStringObject("UNLINK",6);
    shared.rpop = createStringObject("RPOP",4);
    shared.lpop = createStringObject("LPOP",4);
    shared.lpush = createStringObject("LPUSH",5);
    shared.rpoplpush = createStringObject("RPOPLPUSH",9);
    shared.lmove = createStringObject("LMOVE",5);
    shared.blmove = createStringObject("BLMOVE",6);
    shared.zpopmin = createStringObject("ZPOPMIN",7);
    shared.zpopmax = createStringObject("ZPOPMAX",7);
    shared.multi = createStringObject("MULTI",5);
    shared.exec = createStringObject("EXEC",4);
    shared.hset = createStringObject("HSET",4);
    shared.srem = createStringObject("SREM",4);
    shared.xgroup = createStringObject("XGROUP",6);
    shared.xclaim = createStringObject("XCLAIM",6);
    shared.script = createStringObject("SCRIPT",6);
    shared.replconf = createStringObject("REPLCONF",8);
    shared.pexpireat = createStringObject("PEXPIREAT",9);
    shared.pexpire = createStringObject("PEXPIRE",7);
    shared.persist = createStringObject("PERSIST",7);
    shared.set = createStringObject("SET",3);
    shared.eval = createStringObject("EVAL",4);

    /* Shared command argument */
    shared.left = createStringObject("left",4);
    shared.right = createStringObject("right",5);
    shared.pxat = createStringObject("PXAT", 4);
    shared.time = createStringObject("TIME",4);
    shared.retrycount = createStringObject("RETRYCOUNT",10);
    shared.force = createStringObject("FORCE",5);
    shared.justid = createStringObject("JUSTID",6);
    shared.entriesread = createStringObject("ENTRIESREAD",11);
    shared.lastid = createStringObject("LASTID",6);
    shared.default_username = createStringObject("default",7);
    shared.ping = createStringObject("ping",4);
    shared.setid = createStringObject("SETID",5);
    shared.keepttl = createStringObject("KEEPTTL",7);
    shared.absttl = createStringObject("ABSTTL",6);
    shared.load = createStringObject("LOAD",4);
    shared.createconsumer = createStringObject("CREATECONSUMER",14);
    shared.getack = createStringObject("GETACK",6);
    shared.special_asterick = createStringObject("*",1);
    shared.special_equals = createStringObject("=",1);
    shared.redacted = makeObjectShared(createStringObject("(redacted)",10));

    for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
        shared.integers[j] =
            makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
        initObjectLRUOrLFU(shared.integers[j]);
        shared.integers[j]->encoding = OBJ_ENCODING_INT;
    }
    for (j = 0; j < OBJ_SHARED_BULKHDR_LEN; j++) {
        shared.mbulkhdr[j] = createObject(OBJ_STRING,
            sdscatprintf(sdsempty(),"*%d\r\n",j));
        shared.bulkhdr[j] = createObject(OBJ_STRING,
            sdscatprintf(sdsempty(),"$%d\r\n",j));
        shared.maphdr[j] = createObject(OBJ_STRING,
            sdscatprintf(sdsempty(),"%%%d\r\n",j));
        shared.sethdr[j] = createObject(OBJ_STRING,
            sdscatprintf(sdsempty(),"~%d\r\n",j));
    }
    /* The following two shared objects, minstring and maxstring, are not
     * actually used for their value but as a special object meaning
     * respectively the minimum possible string and the maximum possible
     * string in string comparisons for the ZRANGEBYLEX command. */
    shared.minstring = sdsnew("minstring");
    shared.maxstring = sdsnew("maxstring");
}
```


#### 2.2 嵌入式字符串
redisObject在创建字符串对象时，会走如下代码：
```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
- embstr：小于44字节，嵌入式存储，redisObject 和 SDS 一起分配内存，只分配 1 次内存。
- rawstr：大于44字节，redisObject 和 SDS 分开存储，需分配 2 次内存。
- long：整数存储（小于 10000，使用共享对象池存储，但有个前提：Redis 没有设置淘汰策略，详见 object.c 的 tryObjectEncoding 函数）。  
长度是44的原因：
redisObject：16个字节；SDS的sdshdr8（3个字节）+ SDS 字符数组（N 字节 + \0 结束符 1 个字节）
Redis规定嵌入式字符串最大以64字节存储，所以N=64-6(redisObject)-3(sdshr8)-1(\0)，N=44字节。  

注意：  
1.内嵌字符串是只读的，无修改操作的，内嵌字符串一旦进行append等操作，会直接转为raw字符串。  
2.浮点型字符串存储是存成str类型，操作时redis会做内部转换。
3.整数字符在进行拼接字符串操作后也会变为raw类型。

#### 2.3 根据数据量使用不同的内部数据结构
##### 2.3.1 列表的容量变化
如下实现了LPUSH等操作的函数，首次创建list对象时，会条用createListListpackObject,即在开始数据量较小时，先使用listpack存储。

```c
/* Implements LPUSH/RPUSH/LPUSHX/RPUSHX. 
 * 'xx': push if key exists. */
void pushGenericCommand(client *c, int where, int xx) {
    int j;

    robj *lobj = lookupKeyWrite(c->db, c->argv[1]);
    if (checkType(c,lobj,OBJ_LIST)) return;
    if (!lobj) {
        if (xx) {
            addReply(c, shared.czero);
            return;
        }

        lobj = createListListpackObject();
        dbAdd(c->db,c->argv[1],lobj);
    }

    listTypeTryConversionAppend(lobj,c->argv,2,c->argc-1,NULL,NULL);
    for (j = 2; j < c->argc; j++) {
        listTypePush(lobj,c->argv[j],where);
        server.dirty++;
    }

    addReplyLongLong(c, listTypeLength(lobj));

    char *event = (where == LIST_HEAD) ? "lpush" : "rpush";
    signalModifiedKey(c,c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id);
}

robj *createListListpackObject(void) {
    unsigned char *lp = lpNew(0);
    robj *o = createObject(OBJ_LIST,lp);
    o->encoding = OBJ_ENCODING_LISTPACK; // listpack编码
    return o;
}
```
当进行push时，上述的代码中，还会调用listTypeTryConversionAppend函数，进行list转换
```c
/* This is just a wrapper for listTypeTryConversionRaw() that is
 * able to try conversion before adding elements to the list. */
void listTypeTryConversionAppend(robj *o, robj **argv, int start, int end,
                                 beforeConvertCB fn, void *data)
{
    listTypeTryConversionRaw(o, LIST_CONV_GROWING, argv, start, end, fn, data);
}

/* Check if the list needs to be converted to appropriate encoding due to
 * growing, shrinking or other cases.
 *
 * 'lct' can be one of the following values:
 * LIST_CONV_AUTO      - Used after we built a new list, and we want to let the
 *                       function decide on the best encoding for that list.
 * LIST_CONV_GROWING   - Used before or right after adding elements to the list,
 *                       in which case we are likely to only consider converting
 *                       from listpack to quicklist.
 *                       'argv' is only used in this case to calculate the size
 *                       of a number of objects that will be added to list.
 * LIST_CONV_SHRINKING - Used after removing an element from the list, in which case we
 *                       wanna consider converting from quicklist to listpack. When we
 *                       know we're shrinking, we use a lower (more strict) threshold in
 *                       order to avoid repeated conversions on every list change. */
static void listTypeTryConversionRaw(robj *o, list_conv_type lct,
                                     robj **argv, int start, int end,
                                     beforeConvertCB fn, void *data)
{
    if (o->encoding == OBJ_ENCODING_QUICKLIST) {
        if (lct == LIST_CONV_GROWING) return; /* Growing has nothing to do with quicklist */
        listTypeTryConvertQuicklist(o, lct == LIST_CONV_SHRINKING, fn, data);
    } else if (o->encoding == OBJ_ENCODING_LISTPACK) {
        if (lct == LIST_CONV_SHRINKING) return; /* Shrinking has nothing to do with listpack */
        listTypeTryConvertListpack(o, argv, start, end, fn, data);
    } else {
        serverPanic("Unknown list encoding");
    }
}

/* Check the length and size of a number of objects that will be added to list to see
 * if we need to convert a listpack to a quicklist. Note that we only check string
 * encoded objects as their string length can be queried in constant time.
 *
 * If callback is given the function is called in order for caller to do some work
 * before the list conversion. */
static void listTypeTryConvertListpack(robj *o, robj **argv, int start, int end,
                                       beforeConvertCB fn, void *data)
{
    serverAssert(o->encoding == OBJ_ENCODING_LISTPACK);

    size_t add_bytes = 0;
    size_t add_length = 0;

    if (argv) {
        for (int i = start; i <= end; i++) {
            if (!sdsEncodedObject(argv[i]))
                continue;
            add_bytes += sdslen(argv[i]->ptr);
        }
        add_length = end - start + 1;
    }

    if (quicklistNodeExceedsLimit(server.list_max_listpack_size,
            lpBytes(o->ptr) + add_bytes, lpLength(o->ptr) + add_length))
    {
        /* Invoke callback before conversion. */
        if (fn) fn(data);

        quicklist *ql = quicklistNew(server.list_max_listpack_size, server.list_compress_depth);

        /* Append listpack to quicklist if it's not empty, otherwise release it. */
        if (lpLength(o->ptr))
            quicklistAppendListpack(ql, o->ptr);
        else
            lpFree(o->ptr);
        o->ptr = ql;
        o->encoding = OBJ_ENCODING_QUICKLIST;
    }
}
```
在redis.conf配置文件中，有如下配置可以配置list以listpack存储数据的最大空间：
```c
# Lists are also encoded in a special way to save a lot of space.
# The number of entries allowed per internal list node can be specified
# as a fixed maximum size or a maximum number of elements.
# For a fixed maximum size, use -5 through -1, meaning:
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
# Positive numbers mean store up to _exactly_ that number of elements 正整数为存储的个数配置
# per list node.
# The highest performing option is usually -2 (8 Kb size) or -1 (4 Kb size),
# but if your use case is unique, adjust the settings as necessary.
list-max-listpack-size -2
```
当超过上述配置时，list内部会把数据从listpack转为quicklist

##### 2.3.2 其他配置
针对set\hash\zset，也有相关配置
set
```c
# Sets have a special encoding when a set is composed
# of just strings that happen to be integers in radix 10 in the range
# of 64 bit signed integers.
# The following configuration setting sets the limit in the size of the
# set in order to use this special memory saving encoding.
set-max-intset-entries 512

# Sets containing non-integer values are also encoded using a memory efficient
# data structure when they have a small number of entries, and the biggest entry
# does not exceed a given threshold. These thresholds can be configured using
# the following directives.
set-max-listpack-entries 128
set-max-listpack-value 64
```
```c
/* Factory method to return a set that *can* hold "value". When the object has
 * an integer-encodable value, an intset will be returned. Otherwise a listpack
 * or a regular hash table.
 *
 * The size hint indicates approximately how many items will be added which is
 * used to determine the initial representation. */
robj *setTypeCreate(sds value, size_t size_hint) {
    if (isSdsRepresentableAsLongLong(value,NULL) == C_OK && size_hint <= server.set_max_intset_entries)
        return createIntsetObject();
    if (size_hint <= server.set_max_listpack_entries)
        return createSetListpackObject();

    /* We may oversize the set by using the hint if the hint is not accurate,
     * but we will assume this is acceptable to maximize performance. */
    robj *o = createSetObject();
    dictExpand(o->ptr, size_hint);
    return o;
}
```

zset
```c
# Similarly to hashes and lists, sorted sets are also specially encoded in
# order to save a lot of space. This encoding is only used when the length and
# elements of a sorted set are below the following limits:
zset-max-listpack-entries 128
zset-max-listpack-value 64
```
```c
/* Factory method to return a zset.
 *
 * The size hint indicates approximately how many items will be added,
 * and the value len hint indicates the approximate individual size of the added elements,
 * they are used to determine the initial representation.
 *
 * If the hints are not known, and underestimation or 0 is suitable. 
 * We should never pass a negative value because it will convert to a very large unsigned number. */
robj *zsetTypeCreate(size_t size_hint, size_t val_len_hint) {
    if (size_hint <= server.zset_max_listpack_entries &&
        val_len_hint <= server.zset_max_listpack_value)
    {
        return createZsetListpackObject();
    }

    robj *zobj = createZsetObject();
    zset *zs = zobj->ptr;
    dictExpand(zs->dict, size_hint);
    return zobj;
}
```

hash
```c
# Hashes are encoded using a memory efficient data structure when they have a
# small number of entries, and the biggest entry does not exceed a given
# threshold. These thresholds can be configured using the following directives.
hash-max-listpack-entries 512
hash-max-listpack-value 64
```
```c
int hashTypeSet(robj *o, sds field, sds value, int flags) {
    int update = 0;

    /* Check if the field is too long for listpack, and convert before adding the item.
     * This is needed for HINCRBY* case since in other commands this is handled early by
     * hashTypeTryConversion, so this check will be a NOP. */
    if (o->encoding == OBJ_ENCODING_LISTPACK) {
        if (sdslen(field) > server.hash_max_listpack_value || sdslen(value) > server.hash_max_listpack_value)
            hashTypeConvert(o, OBJ_ENCODING_HT);
    }
    
    if (o->encoding == OBJ_ENCODING_LISTPACK) {
        unsigned char *zl, *fptr, *vptr;

        zl = o->ptr;
        fptr = lpFirst(zl);
        if (fptr != NULL) {
            fptr = lpFind(zl, fptr, (unsigned char*)field, sdslen(field), 1);
            if (fptr != NULL) {
                /* Grab pointer to the value (fptr points to the field) */
                vptr = lpNext(zl, fptr);
                serverAssert(vptr != NULL);
                update = 1;

                /* Replace value */
                zl = lpReplace(zl, &vptr, (unsigned char*)value, sdslen(value));
            }
        }

        if (!update) {
            /* Push new field/value pair onto the tail of the listpack */
            zl = lpAppend(zl, (unsigned char*)field, sdslen(field));
            zl = lpAppend(zl, (unsigned char*)value, sdslen(value));
        }
        o->ptr = zl;

        /* Check if the listpack needs to be converted to a hash table */
        if (hashTypeLength(o) > server.hash_max_listpack_entries)
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        dict *ht = o->ptr;
        dictEntry *de, *existing;
        sds v;
        if (flags & HASH_SET_TAKE_VALUE) {
            v = value;
            value = NULL;
        } else {
            v = sdsdup(value);
        }
        de = dictAddRaw(ht, field, &existing);
        if (de) {
            dictSetVal(ht, de, v);
            if (flags & HASH_SET_TAKE_FIELD) {
                field = NULL;
            } else {
                dictSetKey(ht, de, sdsdup(field));
            }
        } else {
            sdsfree(dictGetVal(existing));
            dictSetVal(ht, existing, v);
            update = 1;
        }
    } else {
        serverPanic("Unknown hash encoding");
    }

    /* Free SDS strings we did not referenced elsewhere if the flags
     * want this function to be responsible. */
    if (flags & HASH_SET_TAKE_FIELD && field) sdsfree(field);
    if (flags & HASH_SET_TAKE_VALUE && value) sdsfree(value);
    return update;
}
```