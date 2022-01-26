# STL 源码剖析

## 1、 第一级空间配置器

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

// 注意，以下针对原生指针设计 __type_traits 偏特化版本
// 原生指针亦被视为一种标量型别
template <class T>
struct __type_traits<T*> {
    typedef __true_type has_trivial_default_constructor;
    typedef __true_type has_trivial_copy_constructor;
    typedef __true_type has_trivial_assignment_operator;
    typedef __true_type has_trivial_destructor;
    typedef __true_type is_POD_type;
};
```

## 4、C++ Primer上的类vector类StrVec

```C++
class StrVec {
public:
    StrVec(): 
        elements (nullptr), first_free (nullptr), cap (nullptr) { }
    StrVec (const StrVec&);
    StrVec& operator= (const StrVec&);
    StrVec (StrVec&&);
    StrVec& operator= (StrVec&&);
    ~StrVec();
    void push_back (const string &s);
    size_t size() const { return first_free - elements; }
    size_t capacity() const { return cap - elements; }
    std::string* begin () { return elemnts; }
    std::string* end() { return first_free; }
private:
    static std::allocator<std::string> alloc;
    void chk_n_alloc()
    {
        if (size() == capacity())
            reallocate();
    }
    std::pair<std::string*, std::string*> alloc_n_cpy
    (const std::string*, const std::string*);
    void free();
    void reallocate();
    std::string *elements;
    std::string *first_free;
    std::string* cap;
};

void StrVec::push_back (const std::string &s) {
    chk_n_alloc();
    alloc.construct (first_free++, s);
}

pair<string*, string*>
StrVec::alloc_n_cpy (const string *b, const string *e) {
    auto data = alloc.allocate (e - b);
    return {data, uninitialized_copy (b, e, data)};
}

const StrVec::free() {
    if (elements) {
        for (auto p = first_free; p != elements; )
            alloc.destroy (--p);
        alloc.deallocate (first_free, capacity());
    }
}

StrVec::StrVec (const StrVec &s) {
    auto newdata = alloc_n_copy (s.begin(), s.end());
    elements = newdata.first;
    first_free = cap = newdata.second;
}

StrVec::~StrVec()
{ free(); }

StrVec& StrVec::operator= (const StrVec &rhs) {
    auto newdata = alloc_n_cpy (rhs.begin(), rhs.end());
    free();
    elements = newdata.first;
    first_free = cap = newdata.second;
    return *this;
}

void StrVec::reallocate () {
    auto newcapacity = size() ? 2 * size() : 1;
    auto newdata = alloc.allocate (newcapacity);
    auto dest = newdata;
    auto elem = elements;
    for (size_t i = 0; i != size(); ++i)
        alloc.construct (dest++, std::move (*eleme++));
    free();
    elements = newdata;
    first_free = dest;
    cap = elements + newcapacity;
}

void StrVec::reallocate()
{
    auto newcapacity = size() ? 2 * size() : 1;
    auto first = alloc.allocate (newcapacity);
    auto last = uninitialized_copy (make_move_iterator(begin()),
                       make_move_iterator(end()), first);
    free();
    elements = first;
    first_free = last;
    cap = elements + newcapacity;
}

StrVec::StrVec (StrVec&& s) noexcept
    : elements (s.elements), first_free (s.first_free), cap (s.cap) {
        s.elements = s.first_free = s.cap = nullptr;
}

StrVec& StrVec::operator= (StrVec&& rhs) noexcept {
    if (this != &rhs) {
        free();
        elements = rhs.elements;
        first_free = rhs.first_free;
        cap = rhs.cap;
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}
```

### 更新的三五法则

+ 与拷贝操作不同，编译器根本不会为某些类合成移动操作。==特别是，如果一个类定义了自己的拷贝构造函数、拷贝复制运算符或者析构函数，编译器就不为它合成移动构造函数和移动复制运算符了。==
+ 如果一个类没有移动操作，通过正常的函数匹配，类会使用对应的拷贝操作来代替移动操作。
+ 只有当一个类没有定义任何自己版本的拷贝控制成员，且类的每个非static数据成员都可以移动时，编译器才会为它合成移动构造函数或移动复制运算符。
+ **==只有当一个类没有定义任何自己版本的拷贝控制成员，且它的所有数据成员都能移动构造或移动赋值时，编译器才会为它合成移动构造函数或移动赋值运算符。==**
+ ==*与拷贝操作不同，移动操作永远不会隐式定义为删除的函数。* **但是，如果我们显式地要求编译器生成`=default`的移动操作，且编译器不能移动所有成员，则编译器会将移动操作定义为删除的函数。**==
+ *与拷贝构造函数不同，移动构造函数被定义为删除的函数的条件是：有类成员定义了自己的拷贝构造函数且未定义移动构造函数，或者是有类成员未定义自己的拷贝构造函数且编译器不能为其合成移动构造函数。移动复制运算符的情况类似。*
+ *如果有类成员的移动构造函数或移动赋值运算符被定义为删除的或或是不可访问的，则类的移动构造函数或移动赋值运算符被定义为删除的。*
+ *类似拷贝构造函数，如果类的析构函数被定义为删除的或不可访问的，则类的移动构造函数被定义为删除的。*
+ *类似拷贝赋值运算符，如果有类成员是`const`的或是引用，则类的移动赋值运算符被定义为删除的。*
+ ==**如果类定义了一个移动构造函数和/或一个移动赋值运算符，则该类的合成拷贝构造函数和拷贝赋值运算符会被定义为删除的。**==
+ ==*定义了一个移动构造函数或移动赋值运算符的类必须也定义自己的拷贝操作。**否则，这些成员默认地被定义为删除的。***==
+ ==如果一个类没有移动构造函数，函数匹配规则保证该类型的对象会被拷贝，即使我们试图调用`move`来移动它们时也是如此。==
+ **区分移动和拷贝的重载函数通常有一个版本接受`const T&`，而另一个版本接受一个`T&&`。**
```C++
class StrVec {
public:
    void push_back (const std::string&);
    void push_back (std::string&&);
};

void StrVec::push_back (const std::string& s) {
    chk_n_alloc();
    alloc.construct (first_free++, s);
}

void StrVec::push_back (std::string&& s) {
    chk_n_alloc();
    alloc.construct (first_free++, std::move(s));
}
```

### 右值和左值引用成员函数

+ 我们指出`this`的左值/右值属性的方式与定义`const`成员函数相同，即，在参数列表后放置一个`引用限定符`：
```C++
class Foo {
public:
    Foo &operator= (const Foo&) &;  // 只能向可修改的左值赋值    
};
Foo &Foo::operator= (const Foo &rhs) &
{
    // 执行将rhs赋予本对象所需的工作
    return *this;
}
```
+ 引用限定符可以是`&`或`&&`，分别指出`this`可以指向一个左值或右值。
+ ==类似`const`限定符，引用限定符只能用于（`非static`）成员函数，且必须同时出现在函数的声明和定义中。==
+ ==**一个函数可以同时用`const`和`引用限定`。在此情况下，引用限定符必须跟随在`const`限定符之后。**==

### 重载和引用函数

```C++
class Foo {
public:
    Foo sorted() &&;
    Foo sorted() const &;
private:
    vector<int> data;
};

Foo Foo::sorted() &&
{
    sort (data.begin(), data.end());
    return *this;
}

Foo Foo::sorted() const & {
    Foo ret (*this);
    sort (ret.data.begin(), ret.data.end());
    return ret;
}
```
+ ==如果一个成员函数有引用限定符，则具有相同参数列表的所有版本都必须有引用限定符。==

## 重载运算与类型转换

1. ==除了重载的函数调用运算符`operator()`之外，其他重载运算符不能含有默认是惨。==
2. **当一个重载的运算符是成员函数时，`this`绑定到左侧运算对象。==成员运算符函数的（显式）参数数量比运算对象的数量少一个。==**
3. ==对一个运算符函数来说，它或者是类的成员，或者至少含有一个类类型的参数。==
4. 重载的运算符的优先级和结合律与对应的内置运算符保持一致。
5. ==**通常情况下，不应该载逗号、取地址、逻辑与和逻辑或运算符。**==
6. + *赋值（=）、下标（[ ]）、调用（()）和成员访问箭头（->）运算符必须是成员。*
   + 复合赋值运算符一般来说应该是成员，但并非必须，这一点与赋值运算符略有不同。
   + ==改变对象状态的运算符或者与给定类型密切相关的运算符，如递增、递减和解引用运算符，通常应该是成员。==
   + 具有对称性的运算符可能转换任意一端的运算对象，例如算术、相等性、关系和位运算符等，因此它们通常应该是普通的非成员函数。
   + ==如果我们想提供含有类对象的混合类型表达式，则运算符必须定义成非成员函数。==
#### 重载输出运算符`<<`
  
1. 通常情况下输出运算符的第一个形参是一个非常量`ostream`对象的引用。
2. 第二个形参一般来说是一个常量的引用，该常量是我们想要打印的类类型。
3. 为了与其他输出运算符一致，`operator<<`一般要返回它的`ostream`形参。
```C++
ostream &operator<< (ostream &os, const Sales_data &item) {
    os << item.isbn() << " " << item.units_sold << " "
       << item.revenue << " " << item.avg_price();
    return os;
}
```
4. 输出运算符尽量减少格式化操作
5. **==输入输出运算符必须是非成员函数==**
6. ==`IO`运算符通常需要读写类的非公有数据成员，所以`IO`运算符一般被声明为友元。==
#### 重载输入运算符
1. 输入运算符的第一个形参是运算符将要读取的流的引用，第二个形参时将要读入到的（非常量）对象的引用。

```C++
istream &operator>> (istream &is, Sales_data &item) {
    double price;
    is >> item.bookNo >> item.units_sold >> price;
    if (is)
        item.revenue = item.units_sold * price;
    else
        item = Sales_data();
    return is;
}
```
2. ==输入运算符必须处理输入可能失败的情况，而输出运算符不需要。==
3. ==当读取操作发生错误时，输入运算符应该负责从错误中恢复。==

### 算术和关系运算符
1. 通常情况下，我们把算术和关系运算符定义成非成员函数以允许对左侧或右侧的运算对象进行转换。因为这些运算符一般不需要改变运算对象的状态，所以形参都是常量的引用。
2. ==如果类定义了算术运算符，则它一般也会定义一个对应的复合赋值运算符。**此时，最有效的方式是使用复合赋值来定义算术运算符。**==
```C++
Sales_data 
operator+ (const Sales_data &lhs, const Sales_data &rhs) {
    Sales_data sum - lhs;
    sum += rhs;
    return sum;
}
```
#### 相等运算符
```C++
bool operator==(const Sales_data &lhs, const Sales_data &rhs) {
    return lhs.isbn() == rhs.isbn() &&
           lhs.units_sold == rhs.units_sold &&
           lhs.revenue == rhs.revenue;
}
bool operator!= (const Sales_data &lhs, const Sales_data &rhs) {
    return !(lhs == rhs);
}
```
+ 相等运算符和不相等运算符中的一个应该把工作委托给另外一个，这意味着其中一个运算符应该负责实际比较对象的工作，而另一个运算符则只是调用那个真正工作的运算符。

```C++
bool operator== (const StrBlob &lhs, const StrBlob &rhs) {
    return lhs.data == rhs.data;
}
bool operator!= (const StrBlob &lhs, const StrBlob &rhs) {
    return !(lsh == rhs);
}

bool operator== (const StrBlobPtr &lhs, const StrBlobPtr &rhs) {
    auto l = lhs.wptr.lock(), r = rhs.wptr.lock();
    if (l == r)
        return (!r || lhs.curr == rhs.curr);
    else
        return false;
}
bool operator!= (const StrBlobPtr &lhs, const StrBlobPtr &rhs) {
    return !(lhs == rhs);
}

bool operator== (const StrVec &lhs, const StrVec &rhs) {
    if (lhs.size() != rhs.size())
        return false;
    
    for (auto itr1 = lhs.begin(), itr2 = rhs.begin(); itr1 != lhs.end() && itr2 != rhs.end(); itr1++, itr2++) {
        if (*itr1 != *itr2)
            return false;
    }
}
bool operator!= (const StrVec &lhs, const StrVec &rhs) {
    return !(lhs == rhs);
}
```
#### 关系运算符
1. 通常情况下关系运算符应该：
   + 定义顺序关系，令其与关联容器中对关键字的要求一致；并且
   + 如果类同时也含有`==`运算符的话，则定义一种关系令其与`==`保持一致。特别是，如果两个对象时`!=`的，那么一个对象应该`<`另外一个。
2. 如果存在唯一一种逻辑可靠的`<`定义，则应该考虑为这个类定义`<`运算符。如果类同时还包含`==`，则当且仅当`<`的定义和`==`产生的结果一致时才定义`<`运算符。
```C++
bool operator< (const StrBlob &s1, const StrBlob &s2) {
    return *s1.data == *s2.data;
}
bool operator<= (const StrBlob &s1, const StrBlob &rhs) {
    return *s1.data <= *s2.data;
}
bool operator> (const StrBlob &s1, const StrBlob &s2) {
    return *s1.data > *s2.data;
}
bool operator>= (const StrBlob &s1, const StrBlob &s2) {
    return *s1.data >= *s2.data;
}

bool operator< (const StrBlobPtr &lhs, const StrBlobPtr &rhs) {
    auto l = lhs.wptr.lock(), r = rhs.wptr.lock();
    if (l == r) 
        if (!r)
            return false;
        return (lhs.curr < rhs.curr);
    else
        return false;
}

bool operator<= (const StrBlobPtr &lhs, const StrBlobPtr &rhs) {
    auto l = lhs.wptr.lock(), rhs.wptr.lock();
    if (l == r) 
        return (!r || lhs.curr <= lhs.curr>);
    else
        return false;
}

bool operator> (const StrBlobPtr &lhs, const StrBlobPtr &rhs) {
    auto l = lhs.wptr.lock(), r = rhs.wptr.lock();
    if (l == r) 
        if (!r || )
            return false;
        return lhs.curr == rhs.curr;
    else
        return false;
}
bool operator>= (const StrBlobPtr &lhs, const StrBlobPtr &rhs) {
    auto l = lhs.wptr.lock(), r = rhs.wptr.lock();
    if (l == r) 
        return (!r || lhs.curr >= rhs.curr);
    else
        return false;
}
```
### 赋值运算符

```C++
StrVec& StrVec::operator= (const std::initializer_list<string> il) {
    auto data = alloc_n_cpy (il.begin(), il.end());
    free();
    elements = data.first;
    first_free = cap = data.second;
    return *this;
}
```
+ ==和拷贝赋值及移动赋值运算符一样，其他重载的赋值运算符也必须先释放当前内存空间，再创建一片新空间。==
+ **==不同之处是，这个运算符无须检查对象向自身的赋值，这是因为它的形参`initializer_list<string>`确保`il`与`this`所指的不是同一个对象。==**
+ *我们可以重载赋值运算符。不论形参的类型是什么，==赋值运算符都必须定义为成员函数。==*

### 下标运算符
+ ==**下标运算符必须是成员函数。**==
+ *如果一个类包含下标运算符，则它通常会定义两个版本：一个返回普通引用，另一个是类的常量成员并且返回常量引用。*
```C++
class StrVec {
public:
    std::string& operator[] (std::size_t n)
    { return elements[n]; }   
    const std::string& operator[] (std::size_t n) const 
    { return elements[n]; }
private:
    std::string* elements;
};
```
--------------------------
### 递增和递减运算符
+ ==`C++`语言并不要求递增和递减运算符必须是类的成员，但是因为它们改变的正好是所操作对象的状态，所以建议将其设定为成员函数。==
+ **定义递增和递减运算符的类应该同时定义前置版本和后置版本。这些运算符通常应该被定义成类的成员。**
+ ==**为了与内置版本保持一致，前置运算符应该返回递增或递减后对象的引用。**==
```C++
StrBlobPtr &StrBlobPtr::operator++()
{
    check (curr, "increment past end of StrBlobPtr");
    ++curr;
    return *this;
}

StrBlobPtr &StrBlobPtr::operator--()
{
    --curr;
    check (curr, "decrement past begin of StrBlobPtr");
    return *this;
}
```
+ ==为了与内置版本保持一致，后值运算符应该返回对象的原值（递增或递减之前的值），返回的形式是一个值而非引用。==
```C++
StrBlobPtr StrBlobPtr::operator++(int)
{
    StrBlobPtr ret = *this;
    ++*this;
    return ret;
}

StrBlobPtr StrBlobPtr::operator--(int)
{
    StrBlobPtr ret = *this;
    --*this;
    return ret;
}
```

### 成员访问运算符
```C++
class StrBlobPtr {
public:
    std::string& operator*() const
    {
        auto p = check (curr, "dereference past end");
        return (*p)[curr];
    }
    std::string* operator->() const
    {
        return &this->operator*();
    }
};
```
+ ==**箭头运算符必须是类的成员。解引用运算符通常也是类的成员，尽管并非必须如此。**==

### 函数调用运算符
+ 函数调用运算符必须是成员函数。一个类可以定义多个不同版本的调用运算符，相互之间应该在参数数量或类型上有所区别。
+ *如果类定义了调用运算符，则该类的对象称作函数对象。

#### 含有状态的函数对象类

+ 函数对象类通常含有一些数据成员，这些成员被用于定制调用运算符中的操作。

```C++
class PrintString {
public:
    PrintString (ostream &o = cout, char c = ' '):
        os (o), sep (c) { }
    void operator()(const string &s) const 
    { os << s << sep; }
private:
    ostream &os;
    char sep;
};
```
#### `lambda`是函数对象

+ 当我们编写了一个`lambda`后，编译器将该表达是翻译成一个未命名类的未命名对象。在`lambda`表达式产生的类中含有一个重载的函数调用运算符。
  
```C++
stable_sort (words.begin(), word.send(),
             [] (const string &a, const string &b)
             { return s1.size() < s2.size()});

class ShorterString {
public:
    bool operator() (cnnst string &s1, const string &s2) const
    { return s1.size() < s2.size(); }
};
```
+ 默认情况下`lambda`不能改变它所捕获的变量。因此在默认情况下，有`lambda`产生的类当中的函数调用运算符是一个`const`成员函数。如果`lambda`被声明为可变的，则调用运算符就不是`const`的了。

#### 表示`lambda`及相应捕获行为的类

+ ==通过值捕获的变量被拷贝到`lambda`中。因此，这种`lambda`产生的类必须为每个值捕获的变量建立对应的数据成员，同时创建构造函数，令其使用捕获的变量的值来初始化数据成员。==
```C++
auto wc = find_if (words.begin(), words.end(),
                [](const string &a)
                { return a.size() >= sz;});

class SizeComp {
public:
    SizeComp(size_t n): sz (n) { }
    bool operator()(const string &s) const 
    {return s.size() >= sz;}
private:
    size_t sz;
};
```
+ *==`lambda`表达式产生的类不含默认构造函数、赋值运算符及默认析构函数；它是否含有默认的拷贝/移动构造函数则通常要视捕获的数据成员类型而定。==*

### 标准库定义的函数对象
```C++
plus<Type>          equal_to<Type>      logical_and<Type>
minus<Type>         not_equal_to<type>  logical_or<Type>
multiplies<Type>    greater<Type>       logical_not<Type>
divides<type>       greater_equal<Type> 
modulus<Type>       less<Type>
negate<Type>        less_equal<Type>
```

### 可调用对象与`function`

+ ==`C++`语言中有几种可调用的对象：函数、函数指针、`lambda`表达式、`bind`创建的对象以及重载了函数调用运算符的类。==

#### 标准库`function`类型

```C++
map<sring, function<int(int, int)>> binops;
// 我们能把所有可调用对象，包括函数指针、lambda或函数对象在哪内，都添加到这个map中

map<string, function<int(int, int)>> binops = {
    {"+", add},
    {"-", std::minus<int>()},
    {"/", divide()},
    {"*", [](int i, int j) { return i * j;}},
    {"%", mod}
};
```

#### 重载的函数与`function`

+ 我们不能（直接）将重载函数的名字入`function`类型的对象中。

#### 类型转换运算符

+ `类型转换运算符`是类的一种特殊成员函数，它负责将一个类类型的值转换成其他类型。类型转换运算符的一般形式如下所示：
`operator type() const;`
+ ==类型转换运算符既没有显式的返回类型，也没有形参，而且必须定义成类的成员函数。类型转换运算符通常不应该改变待转换对象的内容，因此，类型转换运算符一般被定义成`const`成员。==
+ **==一个类型转换函数必须是类的成员函数；它不能声明返回类型，形参列表也必须为空。类型转换函数通常应该是`const`。

#### 定义含有类型转换运算符的类

```C++
class SmallInt {
public:
    SmallInt (int i = 0): val (i)
    {
        if (i < 0 || i > 255)
            throw std::out_fo_range ("Bad SmallInt value");
    }
    operator int() const { return val; }
private:
    std::size_t val;
};
```

#### 显式的类型转换运算符

+ 为了防止这样的一场发生，`C++11`新标准引入了`显式的类型转换运算符`：
```C++
class SmallInt {
public: 
    explicit operator int() const { return val; }
};
```
+ 和显式的构造函数一样，编译器（通常）也不会将一个显式的类型转换运算符用于隐式类型转换。
+ 当类型转换运算符是显式的时，我们也能执行类型转换，不过必需通过显式的强制类型转换才可以。==该规定存在一个例外，即如果表达式被用作条件，则编译器会将显式的类型转换自动应用于它。**换句话说，当表达式出现在下列位置时，显式的类型转换将被隐式地执行：**==
 > + `if`、`while`及`do`语句的条件部分
 > + `for`语句头的条件表达式
 > + 逻辑非运算符（!）、逻辑或运算符（||）、逻辑与运算符（&&）的运算对象
 > + 条件运算符（? :）的条件表达式。

#### 避免有二义性的类型转换

+ 如果类中包含一个或多个类型转换，则必须确保在类类型和目标类型之间只存在唯一一种转换方式。否则的话，我们编写的代码将很可能会具有二义性。
+ ==**不要令两个类执行相同的类型转换：如果`Foo`类有一个接受`Bar`类对象的构造函数，则不要再`Bar`类中再定义转换目标是`Foo`类的类型转换运算符。**==
+ ==**避免转换目标是内置算术类型的类型转换。如果是当你已经定义了一个转换成算术类型的类型转换时，接下来**==
    > - 不要再定义接受算术类型的重载运算符。
    > - 不要定义转换到多种算术类型的类型转换。
