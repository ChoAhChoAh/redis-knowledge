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


### 三、部分函数源码


### 四、应用