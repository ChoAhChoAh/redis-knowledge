# redis中的数据结构
## 简单动态字符串（SDS）
### 一、结构
redis的字符串并未使用c的字符串，而是自己实现了SDS结构字符串。  
在《redis设计与实现》中，3.0版本redis的SDS结构为：  
```c
struct sdshdr{
    unsigned int len; /* 记录buf数组中已使用字节的数量 */
    unsigned int free; /* 剩余空间长度 */
    char buf[]; /* 实际存储数据的char数组 */
};
```
但在最新master分支的redis源码中，sds结构变为：
```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* 记录buf数组中已使用字节的数量 */
    uint8_t alloc; /* 分配长度（除开头尾字符） */
    unsigned char flags; /* 低三位是数据结构, 高三位并未使用 */
    char buf[]; /* 实际存储数据的char数组 */
};
```
新版本去除了free属性，并增加了alloc属性代替;新增了flag用于区分预置的集中sds(sdshdr5、sdshdr16、sdshdr32、sdshdr64)。  
### 二、SDS结构的特性：  
1. 兼容部分c字符串函数：SDS字符串遵循了c字符串以空字符“\0”结尾的形式，可复用部分c的字符串操作的函数。
2. 提高字符长度获取速度：相较c的字符串，时间复杂度从O(n)降低为O(1)。  
3. 杜绝缓冲区溢出：SDS结构的相关方法在执行前会进行空间检查和扩容操作。
4. 减少修改字符串时的内存重分配次数：**空间预分配** + **惰性释放** 策略将修改n次字符串的扩容操作数从n次，优化为最大n次。
5. 二进制安全：SDS使用len属性而非空字符来判断字符串结束，因此redis的SDS可以保存文本和二进制数据。
### 三、SDS中一些函数的源码（持续补充中）
#### 3.1 字符串操作函数
sds中实现了多种字符串操作函数，可以去源码中详细查看，当前举例如下：
```c
sds sdscat(sds s, const char *t) {
    return sdscatlen(s, t, strlen(t));
}

sds sdscatsds(sds s, const sds t) {
    return sdscatlen(s, t, sdslen(t));
}

sds sdscpy(sds s, const char *t) {
    return sdscpylen(s, t, strlen(t));
}

sds sdscatrepr(sds s, const char *p, size_t len) {
    s = sdsMakeRoomFor(s, len + 2);
    s = sdscatlen(s,"\"",1);
    while(len--) {
        switch(*p) {
        case '\\':
        case '"':
            s = sdscatprintf(s,"\\%c",*p);
            break;
        case '\n': s = sdscatlen(s,"\\n",2); break;
        case '\r': s = sdscatlen(s,"\\r",2); break;
        case '\t': s = sdscatlen(s,"\\t",2); break;
        case '\a': s = sdscatlen(s,"\\a",2); break;
        case '\b': s = sdscatlen(s,"\\b",2); break;
        default:
            if (isprint(*p))
                s = sdscatlen(s, p, 1);
            else
                s = sdscatprintf(s,"\\x%02x",(unsigned char)*p);
            break;
        }
        p++;
    }
    return sdscatlen(s,"\"",1);
}
```
以上函数存在部分差异，但最终调用的方法为sdscatlen：
```c
/* 
 * s: 源字符串
 * t: 待拼接字符串
 * len: 待拼接字符串长度
 */
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s); /* 获取源字串长度 */

    s = sdsMakeRoomFor(s,len); /* 自动扩容方法 */
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len); /* 拷贝拼接,将t拼接到s */
    sdssetlen(s, curlen+len); /* 更新s长度 */
    s[curlen+len] = '\0'; /* 末尾增加空字符 */
    return s; /* 返回拼接后的字符串 */
}
```
#### 3.2 获取字符串长度函数sdslen
获取字符串长度的函数，可参考下方摘取的源码。  
方法会首选获取sds结构的flags属性，此处redis使用了-1下标这种方式获取flags属性，
根据前文的结构介绍，sds常规使用的结构（除sdshdr5之外），均包含4个属性，flags属性在buf之前。
sds采用了`__attribute__ ((__packed__))`，取消了字节对齐，在结构使用阶段，指针一般指向buf，使用-1相当于之前往前访问了
一个位置，即属性flags。
（[详细可参考此文章中的说明](http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference) ） 
```c
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```
sdslen方法中，还调用了SDS_HDR方法，该方法用于返回类型sds实际类型，用以获取结构的len。(TODO：此处可能解释的不对，待后续重新学习c语言后更正)

#### 3.3 自动扩容函数sdsMakeRoomFor
在上述小节中，sdscatlen函数中调用了自动扩容方法sdsMakeRoomFor，并最终调用了_sdsMakeRoomFor函数，可参考下方代码。  
函数参数中：  
1. s为源字串
2. addlen为增加长度
3. greedy控制是否预分配  

注意点：
1. 下方代码中，若扩容检测发现空间够用（if (avail >= addlen) return s;），则不会扩容。  
当开启预分配（greedy==1）,如果新长度未达到sds最大分配上限（SDS_MAX_PREALLOC，即1MB）,则预分配两倍的新长度。
如果达到最大分配上限，则只追加SDS_MAX_PREALLOC大小的预分配空间。追加后，会进行内存重分配和指针更新等操作。
2. 当因为扩容导致sds类型变化，例如从sdshdr8增长为sdshdr16，则需要拷贝字串、分配空间、释放原有空间和更新指针。
```c
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    reqlen = newlen = (len+addlen);
    assert(newlen > len);   /* Catch size_t overflow */
    if (greedy == 1) {
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;
    }

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    if (oldtype==type) {
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}
```
#### 3.4 字符串裁剪函数sdstrim
通过该函数可知，当进行字符串裁剪时，并未释多余的空间，对应了前文提到的“惰性释放”策略
```c
sds sdstrim(sds s, const char *cset) {
    char *end, *sp, *ep;
    size_t len;

    sp = s;
    ep = end = s+sdslen(s)-1;
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > sp && strchr(cset, *ep)) ep--;
    len = (ep-sp)+1;
    if (s != sp) memmove(s, sp, len);
    s[len] = '\0';
    sdssetlen(s,len);
    return s;
}
```
#### 3.5 空间释放函数sdsfree
函数参考下方截取的源码,最终调用了s_free函数
```c
/* Free an sds string. No operation is performed if 's' is NULL. */
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```
此外，sds_free函数也调用了s_free函数。
TODO:根据进度后续补充空间释放的时机。