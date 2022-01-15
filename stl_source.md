# STL 源码剖析

##1、 第一级空间配置器

```C++
#if 0
# include <new>
# define __THROW_BAD_ALLOC throw bad_alloc
#elif !defined(__THROW_BAD_ALLOC)
# include<iostream.h>
# define __THROW_BAD_ALLOC cerr << "out of memory" << endl; exit(1)
#endif

// malloc-based allocator. 通常比稍后介绍的 default alloc 速度慢
// 一般而言是 thread-safe，并且对于空间的运用比较高效
// 以下是第一级配置器
// 注意，无 "template 型别参数". 至于 “非型别参数” inst，则完全没派上用场
template <int inst>
class _malloc_alloc_template {

private:
    // 以下函数将用来处理内存不足的情况
    // oom : out of memory.
    static void *oom_malloc (size_t);
    static void *oom_realloc (void *, size_t);
    static void (* __malloc_alloc_oom_handler)();

public:
    static void* allocate (size_t n) 
    {
        void *result = malloc (n);  // 第一级配置器直接使用 malloc()
        // 以下无法满足需求时，该用 oom_malloc()
        if (0 == result)
            result = oom_malloc (n);
        return result;
    }

    static void deallocate (void *p, size_t /* n */)
    {
        free (p);
    }

    static void* reallocate (void *p, size_t /* old_sz */, size_t new_sz)
    {
        void* result = realloc (p, new_sz); // 第一级配置器直接使用 realloc()
        // 以下无法满足需求时，该用 oom_realloc()
        if (0 == result)
            oom_realloc (p, new_sz);
            return result;
    }

    // 以下方针 C++ 的 set_new_handler()。换句话说，你可以通过它
    // 指定你自己的 out-of-memory handler
    static void (* set_malloc_handler(void (*f)())) ()
    {
        void (*old)() = __malloc_alloc_oom_handler;
        __malloc_alloc_oom_handler = f;
        return old;
    }
};

// malloc_alloc out-of-memory handling
// 初值为0.有待客端设定
template<int inst>
void (*__malloc_alloc_template<inst>::_malloc_alloc_oom_handler)() = 0;

template<int inst>
void * __malloc_alloc_template<inst>::oom_malloc (size_t n)
{
    void (* my_malloc_handler)();
    void *result;

    for (;;) {
        my_malloc_handler = __malloc_alloc_oom_handler;
        if ( 0 == my_malloc_handler) { __THROW_BAD_ALLOC; }
        (*my_malloc_handler)();
        result = malloc (n);
        if (result) return result;
    }
}

template <int inst>
void * __malloc_alloc_template<inst>::oom_realloc (void *p, size_t n) {
    void (* my_malloc_handler)();
    void *result;

    for (;;) {
        my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == my_malloc_handler) { THROW_BAD_ALLOC; }
        (*my_malloc_handler)();
        reuslt = realloc (p, n);
        if (result) return result;
    }
}

// 注意，以下直接将参数 inst 指定为 0
typedef __malloc_alloc_template<0> malloc_alloc;
```
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
    // 以下函数根据区块大小，决定使用第n号free-list。n从0开始算起
    static size_t FREELIST_INDEX (size_t bytes) {
        return ((bytes) + __ALIGN - 1) / __ALIGN - 1;
    }

    // 返回一个大小为n的对象，并可能加入大小为n的其他区块到free list
    static void *refill (size_t n);
    // 配置一大块空间，可容纳 nobjs 个大小为 “size”的区块
    // 如果配置 nobjs 个区块有所不便，nobjs可能会降低
    static char *chunk_alloc (size_t size, int& nobjs);

    // Chunk allocation state
    static char *start_free;
    static char *end_free;
    static size_t heap_size;

public:
    // n 必须大于0
    static void* allocate (size_t n) {
        obj* volatile *my_free_list;
        obj* result;

        // 大于128就调用第一级配置器
        if (n > (size_t)__MAX_BYTES) {
            return malloc_alloc::allocate (n);
        }

        // 寻找 16 个free list中适当的一个
        my_free_list = free_list + FREELIST_INDEX (n);
        result = *my_free_list;

        if (result == 0) {
            void *r = refill (ROUND_UP(n));
            return r;
        }
        *my_free_list = result->free_list_link;
        return result;
    }

    // p 不可以是 0
    static void deallocate (void *p, size_t n) {
        obj* q = (obj*)p;
        obj* volotile *my_free_list;

        // 大于 128 就调用第一级配置器
        if (n > (size_t)__MAX_BYTES) {
            malloc_alloc::deallocate (p, n);
            return;
        }
        // 寻找对应的 free list
        my_free_list = free_list + FREELIST_INDEX(n);
        // 调整 free list，回收区块
        q->free_list_link = *my_free_list;
        *my_free_list = q;
    }

    static void* reallocate (void* p, size_t old_sz, size_t new_sz);
};

template<bool threads, int inst>
char *__default_alloc_template<threads, inst>::start_free = 0;

template<bool threads, int inst>
char *__default_alloc_template<threads, inst>::end_free = 0;

template<bool threads, int inst> 
size_t __default_alloc_template<threads, inst>::heap_size = 0;

template<bool threads, int inst>
__default_alloc_template<threads, inst>::obj* volatile
__default_alloc_template<threads, inst>::free_list[__NFREELISTS] = 
{0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};

// 返回一个大小为 n 的对象，并且有时候会为适当的 free list 增加节点
// 假设 n 已经适当上调至 8 的倍数
template<bool threads, int inst> 
void*
__default_allc_template<threads, inst>::refill (size_t n) {
    int nobjs = 20;
    // 调用 CHunk_alloc()，尝试取得 nnobjs 个区块作为 Free list 的新节点
    // 注意参数 nobjs 是 pass by reference
    char* chunk = chunk_alloc (n, nobjs);
    obj* volatile *my_free_list;
    obj* result;
    obj *current_obj, *next_obj;
    int i;

    // 如果只获得一个区块，这个区块就分配给调用者用，free list 无新节点
    if (1 == nobjs) return chunk;
     // 否则准备调整 free list，纳入新节点
    my_free_list = free_list + FREELIST_INDEX(n);

   // 以下在 chunk 空间内建立 free list
   result = (obj*)chunk;
   // 以下引导free list指向新配置的空间（取自内存池
   *my_free_list = next_obj = (obj*)((char*)chunk + n);
   // 以下将 free list 的各节点串接起来
   for (i = 1; ; ++i) {    // 从 1 开始，因为第 0 个将返回给客端
       current_obj = next_obj;
       next_obj = (obj*)(current_obj + n);
       if (nobjs - 1 == i) {
          current_obj->free_list_link = 0;
          break;
        } 
        else {
            current_obj->free_list_link = next_obj;
        }
    }
    return result;
}

// 
template<bool threads, int inst>
char*
__default_alloc_template<threads, inst>::chunk_alloc (size_t size, int& nobjs) {
    char* result;
    size_t total_bytes = size * nobjs;
    size_t bytes_left = end_free - start_free; // 内存池剩余空间

    if (bytes_left >= total_bytes) {
        // 内存池剩余空间完全满足需求量
        result = start_free;
        start_free += total_bytes;
        return result;
    }
    else if (bytes_left >= size) {
        // 内存池剩余空间不能完全满足需求量，但足够供应一个（含）以上的区块
        nobjs = bytes_left / size;
        total_bytes = size * nobjs;
        result = start_free;
        start_free += total_bytes;
        return result;
    }
    else {
        //内存池剩余空间连一个区块的大小都无法提供
        bytes_to_get = 2 * total_bytes + ROUND_UP(heap_size >> 4);
        // 以下试着让内存池中的残余零头还有利用价值
        if (bytes_left > 0) {
            // 内存池内还有一些零头，先配给适当的 free list
            // 首先寻找适当的 free list
            obj* volatile *my_free_list = free_list + FREELIST_INDEX(bytes_left);
            // 调整 free list，将内存池中的残余空间编入
            ((obj*)start_free)->free_list_link = *my_free_list;
            *my_free_list = (obj*)start_free;
        }

        // 配置 heap 空间，用来补充内存池
        start_free = (char *)malloc (bytes_to_get);
        if (0 == start_free) {
            // heap 空间不足，malloc() 失败
            int i;
            obj* volatile *my_free_list, *p;
            // 试着监视我们手上拥有的东西。这不会造成伤害。我们不打算尝试配置
            // 较小的区块，因为那在多进程（multi-process）机器上容易导致灾难
            // 以下搜寻适当的 free list
            // 所谓适当是指 ”尚有未用区块，且区块够大“ 之 free list
            for (i = size; i <= __MAX_BYTES; i += __ALIGN) {
                my_free_list = free_list + FREELIST_INDEX(i);
                p = *my_free_list;
                if (0 != p) { // free list 内尚有未用区块
                    // 调整 free list 以释出未用区块
                    *my_free_list = p->free_list_link;
                    start_free = (char*)p;
                    end_free = start_free + i;
                    // 递归调用自己，为了修正 nobjs
                    return chunk_alloc (size, nobjs);
                    // 注意，任何残余零头终将被编入适当的 free list 中备用
                }
            }
            end_free = 0; // 如果出现意外（上穷水尽，到处都没内存可用了）
            // 调用第一级配置器，看看 out-of-memory 机制能否尽点力
            start_free = (char*)malloc_alloc::allocate (bytes_to_get);
            // 这会导致抛出异常（exception），或内存不足的情况获得改善
        }
        heap_size += bytes_to_get;
        end_free = start_free + bytes_to_get;
        // 递归调用自己，为了修正 nobjs
        return chunk_alloc (size, nobjs);
    }
}

// 令第二级分配器的名称为 alloc
typedef __default_alloc_template<false, 0> alloc;
```

## 2、`iterator` 源代码完整重列

```C++
// 节选自 SGI STL <stl_iterator.h>
// 五种迭代器类型
struct input_iterator_tag { };
struct output_iterator_tag { };
struct forward_iterator_tag : public input_iterator_tag  { };
struct bidirectional_iterator_tag : public forward_iterator_tag { };
struct random_access_iterator_tag : public bidirectional_iterator_tag { };

// 为避免写代码时挂一漏万，自行开发的迭代器最好继承自下面这个 std::iterator
template <class Category, 
          class T,
          class Distance = ptrdiff_t,
          class Pointer = T*,
          class Reference = T&>
struct iterator {
    typedef Category        iterator_category;
    typedef T               value_type;
    typedef Distance        difference_type;
    typedef Pointer         pointer;
    typedef Reference       reference;
};

// “榨汁机” traits
template <class Iterator>
struct iterator_traits {
    typedef typename Iterator::iterator_category    iterator_category;
    typedef typename Iterator::difference           difference;
    typedef typename Iterator::value_type           value_type;
    typedef typename Iterator::reference            reference;
    typedef typename Iterator::pointer              pointer;
};

// 针对原生指针(native)而设计的 traits 偏特化版
template <class T
struct iterator_traits<T*> {
    typedef random_access_iterator_tag  iterator_category;
    typedef T                           value_type;
    typedef ptrdiff_t                   difference_type;
    typedef T*                          pointer;
    typedef T&                          reference;
};

// 针对原生之 pointer-to-const 而设计的 traits 偏特化版
template iterator_traits<const T*> {
    typedef random_access_iterator_tag  iterator_category;
    typedef T                           value_type;
    typedef ptrdiff_t                   difference_type;
    typedef T&                          reference;
    typedef T*                          pointer;
};

// 这个函数可以很方便地决定某个迭代器的类型（category）
template <class Iterator>
inline typename iterator_traits<Iterator>::iterator_category
iterator_category (const Iterator&) {
    typedef typename iterator_traits<Iterator>::iterator_category category;
    return category();
}

// 这个函数可以很方便地决定某个迭代器的 distance type
template <class Iterator>
inline typename iterator_traits<Iterator>::difference_type*
distance_type (const Iterator&) {
    return static_cast<typename iterator_traits<Iterator>::difference_type*>(0);
}

// 这个函数可以很方便地决定某个迭代器的 value type
template <class Iterator>
inline typename iterator_traits<Iterator>::value_type*
value_type (const Iterator&) {
    return static_cast<typename iterator_traits<Iterator>::value_type*>(0);
}

// 以下是整组 distance 函数
template <class InputIterator>
inline typename iterator_traits<Iterator>::difference_type
__distance (InputIterator first, InputIterator last, input_iterator_tag) {
    iterator_traits<InputIterator>::difference_type n = 0;
    while (first != last) {
        ++first;
        ++n;
    }
    return n;
}

template <class RandomAccessIterator>
inline typename iterator_traits<RandomAccessIterator>::difference_type
__distance (RandomAccessIterator first, RandomAccessIterator last, random_access_iterator_tag) {
    return last - first;
}

template <class InputIterator>
inline typename iterator_traits<InputIterator>::difference_type
distance (InputIterator first, InputIterator last) {
    typedef typename iterator_traits<Input_iterator>::iterator_category category;
    return __distance (first, last, category());
}

// 以下是整组 advance 函数
template <class InputIterator, class Distance>
inline void __advance (InputIterator& i, Distance n, input_iterator_tag) {
    while (n--)  ++i;
}

template <class BidirectionalIterator, class Distance>
void __advance (BidirectionalIterator& i, Distance n) {
    if (n > 0)
        while (n--)
            ++i;
    else
        while (n++);
            --i;
}

template <class RandomAccessIterator, class Distance>
inline void __advance (RandomAccessIterator& i, Distance n) {
    i += n;
}

template <class InputIterator, class Distance>
void advance (InputIterator& i, Distance n) {
    typedef tyename iterator_traits<InputIterator>::iterator_category category;
    __advance (i, n, category(i));
}
```

## 3、 `__type_traits`

```C++
/*以下针对 C++ 基本型别 char, signed char, unsigned char, short, unsigned short, int, unsigned int, long, unsigned long, float, double, long double 提供特化版本。注意，每一个成员的值都是 __true_type，表示这些型别都可采用最快速方式（例如 memcpy）来进行拷贝（copy）或赋值（assign）操作*/

// 注意，SGI STL <stl_config.h> 将以下出现的 __STL_TEMPLATE_NULL
// 定义为 template<>，

__STL_TEMPLATE_NULL struct __type_traits<char> {
    typedef __true_type has_trivial_default__constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
}

__STL_TEMPLATE_NULL struct __type_traits<signed char> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
}

__STL_TEMPLATE_NULL struct __type_traits<unsigned char> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
}

__STL_TEMPLATE_NULL __type_traits<short> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL __type_traits<unsigned short> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<int> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned int> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<long> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned long> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<float> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<double> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<long double> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};