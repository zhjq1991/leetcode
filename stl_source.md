# STL 源码剖析

## 第一级空间配置器

## 第二级空间配置器

+ 下面是第二级配置器的部分市县内容
```C++
enum {__ALIGN = 8};
enum {__MAX_BYTES = 128};
enum {_NFREELISTS = __MAX_BYTES / __ALIGN};

// 以下是第二级配置器
// 注意，无“template型别参数”， 且第二参数完全没派上用场
// 第一参数用于多线程环境下。
template <bool threads, int inst>
class __default_alloc_template {
private:
    // ROUND_UP() 将 bytes 上调至8的倍数
    static size_t ROUND_UP(size_t bytes) {
        return ((bytes) + __ALIGN - 1) & ~(__ALIGN - 1));
    }
private:
    union obj {
        union obj* free_list_link;
        char client_data[1];
    };

private:
    // 16个free_list
    static obj* volatile free_list[__NFREELISTS];
    // 以下函数根据区块大小，决定使用
}