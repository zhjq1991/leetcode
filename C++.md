1. ==简述C++从代码到可执行二进制文件的过程==

   + 预编译：这个过程主要的处理操作如下：

     （1） 将所有的#define删除，并且展开所有的宏定义

     （2） 处理所有的条件预编译指令，如#if、#ifdef

     （3） 处理#include预编译指令，将被包含的文件插入到该预编译指令的位置。

     （4） 过滤所有的注释

     （5） 添加行号和文件名标识。

   + 编译：这个过程主要的处理操作如下：

     （1） 词法分析：将源代码的字符序列分割成一系列的记号。

     （2） 语法分析：对记号进行语法分析，产生语法树。

     （3） 语义分析：判断表达式是否有意义。

     （4） 代码优化：

     （5） 目标代码生成：生成汇编代码。

     （6） 目标代码优化：

   + 汇编：这个过程主要是将汇编代码转变成机器可以执行的指令。

   + 链接：将不同的源文件产生的目标文件进行链接，从而形成一个可以执行的程序。

     链接分为静态链接和动态链接。

     静态链接，是在链接的时候就已经把要调用的函数或者过程链接到了生成的可执行文件中，就算你在去把静态库删除也不会影响可执行程序的执行；生成的静态链接库，Windows下以.lib为后缀，Linux下以.a为后缀。

     而动态链接，是在链接的时候没有把调用的函数代码链接进去，而是在执行的过程中，再去找要链接的函数，生成的可执行文件中没有函数代码，只包含函数的重定位信息，所以当你删除动态库时，可执行程序就不能运行。生成的动态链接库，Windows下以.dll为后缀，Linux下以.so为后缀。

2. ==`new`、`delete`==

   ```C++
   class NewHandlerHolder {
   public:
     explicit NewHandlerHolder (std::new_handler nh) : handler (nh) {}
     ~NewHandlerHolder()
     {
       std::set_new_handler (handler);
     }
   private:
     std::new_handler handler;
     NewHandlerHolder (const NewHandlerHolder&);
     NewHandlerHolder& operator= (const NewHandlerHolder&);
   };
   
   class Widget {
   public:
     static std::new_handler set_new_handler (std::new_handler p) throw();
     static void *operator new (size_t size) throw (bad_alloc);
   private:
     static std::new_handler currentHandler;
   };
   
   std::new_handler Widget::set_new_handler (std::new_handler p)
   {
     std::new_handler oldHandler = currentHandler;
     currentHandler = p;
     return oldHandler;
   }
   
   void * Widget::operator new (size_t size) throw (std::bad_alloc)
   {
     NewHandlerHolder h (std::set_new_handler (currentHandler));
     return ::operator new (size);
   }
   
   template <typename T>
   class NewHandlerSupport {
   public:
     static std::new_handler set_new_handler (std::new_handler p) throw();
     static void *operator new (size_t size) throw (std::bad_alloc);
     ...
   private:
     std::new_handler currentHandler;
   };
   
   template <typename T>
   std::new_handler 
   NewHandlerSupport<T>::set_new_handler (std::new_handler p) throw()
   {
     std::new_handler oldHandler = currentHandler;
     currentHandler = p;
     return oldHandler;
   }
   
   template <typename T>
   void *
   NewHandlerHolder<T>::operator new (std::size_t size) throw (std::bad_alloc)
   {
     NewHandlerHolder h (std::set_new_handler (currentHandler));
     return ::operator new (size);
   }
   
   template <typename T>
   std::new_handler NewHandlerSupport<T>::currentHandler = 0;
   
   
   class Widget : public NewHandlerSupport<Widget> {
   ....						// 和先前一样，但不必声明
     							// set_new_handler或operator new
   };
   ```

3. **实现一致性`operator new`必得返回正确的值，内存不足时必得调用`new-handling`函数，==必须有对付零内存需求的准备，还需避免不慎掩盖正常形式的`new`。==**

4. `C++`规定，即使客户要求0 bytes，`operator new`也得返回一个合法指针。

   ```C++
   void* operator new (std::size_t size) throw (std::bad_alloc)
   {
     using namespace std;
     if (size == 0) {
       size = 1;
     }
     
     while (true) {
   		尝试分配size bytes;
       if (分配成功)
         return (一个指针，指向分配得来的内存);
       
       //分配失败；找出目前的new-handling函数
       new_handler globalHandler = set_new_handler (0);
       set_new_handler (globalHandler);
       
       if (globalHandler) (*globalHandler)();
       else
         throw std::bad_alloc();
     }
   }
   
   class Base {
   public:
   	static void* operator new (std::size_t size) throw (std::bad_alloc);
     ...
   };
   class Derived: public Base {
   ...
   };
   
   void* Base::operator new (std::size_t size) throw (std::bad_alloc)
   {
   	if (size != sizeof (Base))
       return ::operator new (size);
     
     ...
   }
   ```

5. **`operator delete`情况更简单，你需要记住的唯一事情就是`C++`保证“删除`null`指针永远安全“，所以你必须兑现这项保证。**

   ```C++
   // non-member operator delete的伪码
   void operator delete (void *rawMemory) throw()
   {
   	if (rawMemory == 0) 
       return;
     //现在，归还rawMemory所指的内存；
   }
   ```

   1. **`operator delete`的`member`版本也很简单，只需要多加一个动作检查删除数量。万一你的class专属的`operator new`将大小有误的分配行为转交`::operator delete`执行，你也必须将大小有误的删除行为转交`::operator delete`执行。**

      ```C++
      class Base {
      public:
        static void* operator new (std::size_t size) throw (std::bad_alloc);
        static void operator delete (void* rawMemory, std::size_t size) throw ();
        ...
      };
      
      void Base::operator delete (void *rawMemory, std::size_t size) throw()
      {
        if (rawMemory == 0)
          return;
        if (size != sizeof (Base)) {
          ::operator delete (size);
          return;
        }
        //现在，归还rawMemory所指的内存；
        return;
      }
      ```

6. 写了`placement new`也要写`placement delete` 

   1. 如果`operator new`接受的参数除了一定会有的那个`size_t`之外还有其他，这便是个所谓的`placement new`。

   2. 众多`placement new`版本中特别有用的一个是“接受一个指针指向对象该被构造之处”，那样的`operator new`长相如下：

      `void* operator new(std::size_t, void* pMemory) throw();`
      这个`new`的用途之一是负责在vector的未使用空间上创建对象。它同时也是最早的`placement new`版本。

   3. 类似于`new`的`placement`版本，`operator delete`如果接受额外参数，便称为`placement deletes`。

   4. ==如果一个带额外参数的`operator new`没有“带相同额外参数”的对应版`operator delete`，那么当`new`的内存分配动作需要取消并恢复旧观是就没有任何`operator delete`会被调用。==

      ```C++
      class Widget {
      public:
        ...
        static void* operator new(std::size_t size, std::ostream& logStream) throw (std::bad_alloc);
        static void* operator delete (void* pMemory) throw();
        static void operator delete (void* pMemory, std::ostream& logStream) throw();
        ...
      };
      ```

   5. ==缺省情况下`C++`在global作用域内提供以下形式的`operator delete`:==

      ```C++
      void* operator new (std::size_t) throw (std::bad_alloc);
      void* operator new (std::size_t, void*) throw();
      void* operator new (std::size_t, const std::nothrow_t&) throw();
      ```

   6. ==如果你在`class`内声明任何`operator news`，它会遮掩上述这些形式。除非你的意思就是要阻止`class`的客户使用这些形式，否则请确保它们在你所生成任何定制型`operator new`之外还可用。对于每一个可用的`operator new`也请确定提供对应的`operator delete`。如果你希望这些函数有着平常的行为，只要令你的`class`专属版本调用`global`版本即可。==

   7. ==完成以上所言的一个简单做法是，建立一个`base class`，内含所有正常形式的`new`和`delete`：==

      ```C++
      class StandardNewDeleteForms {
      public:
        // normal new/delete
        static void* operator new (std::size_t size) throw (std::bad_alloc)
        {
          return ::operator new (size);
        }
        
        static void operator delete (void* pMemory) throw()
        {
          ::operator delete (pMemory);
        }
        
        //placement new/delete
        static void* operator new (std::size_t size, void* ptr) throw()
        {
          return ::operator new (size, ptr);
        }
        
        static void operator delete (void* pMemory, void* ptr) throw()
        {
          ::operator delete (pMemory, ptr);
        }
        
        //nothrow new/delete
        static void* operator new (std::size_t size, const std::nothrow_t& nt) throw()
        {
          return ::operator (size, nt);
        }
        
        static void operator delete (void *pMemory, const std::nothrow_t&) throw()
        {
          ::operator delete (pMemory);
        }
      };
      ```

      ==凡是想以自定形式扩充标准形式的客户，可利用继承机制及`using`声明式取得标准形式：==

      ```C++
      class Widget: public StandardNewDeleteForms {
      public:
        using StandardNewDeleteForms::operator new;
        using StandardNewDeleteForms::operator delete;
        
        static void* operator new (std::size_t size, std::ostream& logStream) throw (std::bad_alloc);
        static void operator delete (void* pMemory, std::ostream& logStream) throw();
        ...
      };
      ```

7. ==C++ 中的 new/delete 函数与 malloc/free 的区别？==

   主要是前者会调用相关对象的构造、析构函数，后者不会，前者不需要指定申请/释放的内存的大小，后者需要指定等。

   * ==使用 new 申请内存失败会怎么样？==

     会抛出异常。  

   * ==如果我想使得申请失败后返回一个空指针，需要怎么做？==

     使用 `no throw new`来申请内存。

8. ==构造函数能否声明为虚函数或者纯虚函数，析构函数呢？==

   + 析构函数可以为虚函数，并且一般情况下基类析构函数要定义为虚函数，只有在基类析构函数定义为虚函数时，调用操作符 delete 销毁指向对象的基类指针时，才能准确调用派生类的析构函数（从该级向上按序调用虚函数），才能准确销毁数据。
   + 构造函数不能定义为虚函数。在构造函数中可以调用虚函数，不过此时调用的是正在构造的类中的虚函数，而不是子类的虚函数，因为此时子类尚未构造好。虚函数对应一个 vtable (虚函数表)，类中存储一个 vptr 指向这个 vtable。如果构造函数是虚函数，就需要通过 vtable 调用，可是对象没有初始化就没有 vptr，无法找到 vtable，所以构造函数不能是虚函数。

9. ==为什么父类的析构函数必须是虚函数？==

   + 如果析构函数不被声明成虚函数，那么编译器将实施静态绑定，在删除基类指针时，只会调用基类的析构函数而不会调用派生类析构函数，这样就会造成派生类对象析构不完全，造成内存泄漏。

10. ==TCP 粘包和拆包分别是什么？==

    + 粘包：在进行数据传输时，发送端一次性连续发送多个数据包，TCP 协议将多个数据包打包成一个 TCP 报文发送出去；
    + 拆包：当发送端发送的数据包长度超过一次 TCP 报文所能传输的最大值时，就会将该数据包拆分成多个 TCP 报文分开传输。

11. ==如何解决粘包和拆包问题？==

    + 设置消息定长
    + 头部添加消息长度
    + 消息尾部添加特殊字符进行分割
    + 将消息分为消息头和消息尾。

12. ==介绍下智能指针怎么实现的？==

    ```C++
    ```

13. ==内存结构==

    **从低地址到高地址，一个程序由代码段、数据段、BSS段、堆栈段组成。**  

    1. **代码段：**存放程序**执行代码**的一块内存区域。只读，**不允许修改**，代码段的头部还会包含一些**只读的常量**，如**字符串常量字面值**（注意：**const变量**虽然属于常量，但是本质还是变量，不存储于代码段）。
    2. **数据段data：**存放程序中**已初始化**的**全局变量**和**静态变量**的一块内存区域。
    3. **BSS** 段：存放程序中**未初始化**的**全局变量**和**静态变量**的一块内存区域。
    4. 可执行程序在运行时又会多出两个区域：**堆区**和**栈区。**
       + **堆区：**动态申请内存用。堆从低地址向高地址增长。
       + **栈区：**存储**局部变量**、**函数参数值**。栈从高地址向低地址增长。是一块连续的空间。
    5. 最后还有一个**文件映射区（共享区）**，位于堆和栈之间。

14. ==静态成员函数可以直接访问非静态数据成员吗==
    不能。

    **当调用一个对象的非静态成员函数时，系统会把该对象的起始地址赋给成员函数的this指针。而静态成员函数不属于任何一个对象，因此C++规定静态成员函数没有this指针。既然它没有指向某一对象，也就无法对一个对象中的非静态成员进行访问。**

15. ==`socket`编程了解吗？==

    <img src="https://uploadfiles.nowcoder.com/images/20210329/675098158_1617020573807/586E508F161F26CE94633729AC56C602" alt="socket" style="zoom:80%;" />

    图中展示的交互流程，具体如下所述 ：

    （1）服务器根据地址类型（ ipv4, ipv6 ）、 socket 类型、协议创建 socket。

    （2）服务器为 socket 绑定 IP 地址和端口号。

    （3）服务器 socket 监听端口号请求，随时准备接收客户端发来的连接，这时候服务器的socket 并没有被打开 。

    （4）客户端创建 socket。

    （5）客户端打开 socket，根据服务器 IP 地址和端口号试图连接服务器 socket。

    （6）服务器 socket 接收到客户端 socket 请求，被动打开，开始接收客户端请求，直到客户端返回连接信息 。这时候 socket 进入阻塞状态，所谓阻塞即accept（）方法一直到客户端返回连接信息后才返回，开始接收下一个客户端连接请求 。

    （7）客户端连接成功，向服务器发送连接状态信息 。

    （8）服务器 accept 方法返回，连接成功 。

    （9）客户端向 socket 写入信息 。

    （10）服务器读取信息 。

    （11）客户端关闭 。

    （12）服务器端关闭 。

16. ==*如何避免野指针和内存泄漏*==

    + 野指针：指针初始化为`nullptr`、释放内存后指针指向`nullptr`、访问数组时注意边界条件
    + 内存泄漏：配套使用`new/delete`、使用智能指针、基类析构函数设置为虚函数

17. ==智能指针的实现原理==

    + `auto_ptr`----`new`出来的内存实现自动`delte`
    + `unique_ptr`----实现一指针指向一对象
    + `shared_ptr`----实现多指针指向一对象
    + `weak_ptr`----解决共享指针的循环引用问题

18. ==堆与栈的区别==

    + **申请方式不同**：栈是系统自动分配的，寄存器存在对栈操作的指令， 效率高；

      堆是程序员自己手动分配的，效率没那么高，用起来方便但容易出现问题，容易产生碎片；

    + **响应方式不同**：当前栈空间大小大于等于申请的栈空间系统则分配，否则提示栈溢出；

      当申请堆空间则是遍历空闲[链表](https://www.nowcoder.com/jump/super-jump/word?word=链表)寻找第一个大于等于申请空间的节点，并将多余部分合并回[链表](https://www.nowcoder.com/jump/super-jump/word?word=链表)；

    + **生长方式不同**：栈是从高地址到低地址生长，是一段连续的空间，有预先分配的空间大小限制，默认初始化8M；
      堆是从低地址到高地址生长，是一段不连续的空间，堆的大小取决与虚拟内存的大小；

    + **存储内容不同**：栈用于存储函数的参数与局部变量；

      堆用于存储程序员自定义的内容；

19. ==内联函数和宏函数的区别==
    1、内联函数在编译时展开，而宏在预编译时展开

    2、在编译的时候，内联函数直接被嵌入到目标代码中去，而宏只是一个简单的文本替换。

    3、内联函数可以进行诸如类型安全检查、语句是否正确等编译功能，宏不具有这样的功能。

    4、宏不是函数，而inline是函数

    5、宏在定义时要小心处理宏参数，一般用括号括起来，否则容易出现二义性。而内联函数不会出现二义性。

20. ==`MAC `地址和`IP`地址分别有什么作用==

    + IP地址是IP协议提供的一种统一的地址格式，它为互联网上的每一个网络和每一台主机分配一个逻辑地址，以此来屏蔽物理地址的差异。而MAC地址，指的是物理地址，用来定义网络设备的位置。
    + IP地址的分配是**根据网络的拓扑结构**，而不是根据谁制造了网络设置。若将高效的路由选择方案建立在设备制造商的基础上而不是网络所处的拓朴位置基础上，这种方案是不可行的。
    + 当存在一个附加层的地址寻址时，设备更易于移动和维修。例如，如果一个以太网卡坏了，可以被更换，而无须取得一个新的IP地址。如果一个IP主机从一个网络移到另一个网络，可以给它一个新的IP地址，而无须换一个新的网卡。
    + 无论是局域网，还是广域网中的计算机之间的通信，最终都表现为将数据包从某种形式的链路上的初始节点出发，从一个节点传递到另一个节点，最终传送到目的节点。数据包在这些节点之间的移动都是由**ARP**（Address Resolution Protocol：地址解析协议）负责**将IP地址映射到MAC地址上来完成的**。

21. ==简述 TCP 三次握手和四次挥手的过程==

    + ***三次握手***
      <img src="https://uploadfiles.nowcoder.com/images/20220225/4107856_1645789276099/AB3FC1B1325FA341A39644BA061FA439" alt="三次握手" style="zoom: 50%;" />

      + 第一次握手：建立连接时，客户端向服务器发送SYN包（seq=x），请求建立连接，等待确认

      + 第二次握手：服务端收到客户端的SYN包，回一个ACK包（ACK=x+1）确认收到，同时发送一个SYN包（seq=y）给客户端

      + 第三次握手：客户端收到SYN+ACK包，再回一个ACK包（ACK=y+1）告诉服务端已经收到

      + 三次握手完成，成功建立连接，开始传输数据

    + 四次挥手
      <img src="https://uploadfiles.nowcoder.com/images/20220225/4107856_1645789288423/2F42938B52A4B6494AA9CD8FCE658EBD" alt="四次挥手" style="zoom:50%;" />

      + 客户端发送FIN包（FIN=1)给服务端，告诉它自己的数据已经发送完毕，请求终止连接，此时客户端不发送数据，但还能接收数据
      + 服务端收到FIN包，回一个ACK包给客户端告诉它已经收到包了，此时还没有断开socket连接，而是等待剩下的数据传输完毕
      + 服务端等待数据传输完毕后，向客户端发送FIN包，表明可以断开连接
      + 客户端收到后，回一个ACK包表明确认收到，等待一段时间，确保服务端不再有数据发过来，然后彻底断开连接

22. ### ==说说 TCP 2次握手行不行？为什么要3次==

    + 为了实现可靠数据传输， TCP 协议的通信双方， 都必须维护一个序列号， 以标识发送出去的数据包中， 哪些是已经被对方收到的。 三次握手的过程即是通信双方相互告知序列号起始值， 并确认对方已经收到了序列号起始值的必经步骤
    + 如果只是两次握手， 至多只有连接发起方的起始序列号能被确认， 另一方选择的序列号则得不到确认

23. ### ==简述 TCP 和 UDP 的区别，它们的头部结构是什么样的==

    + TCP协议是有连接的，有连接的意思是开始传输实际数据之前TCP的客户端和服务器端必须通过三次握手建立连接，会话结束之后也要结束连接。而UDP是无连接的
    + TCP协议保证数据按序发送，按序到达，提供超时重传来保证可靠性，但是UDP不保证按序到达，甚至不保证到达，只是努力交付，即便是按序发送的序列，也不保证按序送到。
    + TCP协议所需资源多，TCP首部需20个字节（不算可选项），UDP首部字段只需8个字节。
    + TCP有流量控制和拥塞控制，UDP没有，网络拥堵不会影响发送端的发送速率
    + TCP是一对一的连接，而UDP则可以支持一对一，多对多，一对多的通信。
    + TCP面向的是字节流的服务，UDP面向的是报文的服务。

24. ### ==简述 TCP 连接 和 关闭的状态转移==

    + 状态转换如图所示：

      <img src="https://uploadfiles.nowcoder.com/images/20220225/4107856_1645789338936/2FC8F26DA99E984EF442E4AB1024E75F" alt="状态转换" style="zoom:50%;" />

    + 上半部分是TCP三路握手过程的状态变迁，下半部分是TCP四次挥手过程的状态变迁。

      1. **CLOSED**：起始点，在超时或者连接关闭时候进入此状态，这并不是一个真正的状态，而是这个状态图的假想起点和终点。
      2. **LISTEN**：服务器端等待连接的状态。服务器经过 socket，bind，listen 函数之后进入此状态，开始监听客户端发过来的连接请求。此称为应用程序被动打开（等到客户端连接请求）。
      3. **SYN_SENT**：第一次握手发生阶段，客户端发起连接。客户端调用 connect，发送 SYN 给服务器端，然后进入 SYN_SENT 状态，等待服务器端确认（三次握手中的第二个报文）。如果服务器端不能连接，则直接进入CLOSED状态。
      4. **SYN_RCVD**：第二次握手发生阶段，跟 3 对应，这里是服务器端接收到了客户端的 SYN，此时服务器由 LISTEN 进入 SYN_RCVD状态，同时服务器端回应一个 ACK，然后再发送一个 SYN 即 SYN+ACK 给客户端。状态图中还描绘了这样一种情况，当客户端在发送 SYN 的同时也收到服务器端的 SYN请求，即两个同时发起连接请求，那么客户端就会从 SYN_SENT 转换到 SYN_REVD 状态。
      5. **ESTABLISHED**：第三次握手发生阶段，客户端接收到服务器端的 ACK 包（ACK，SYN）之后，也会发送一个 ACK 确认包，客户端进入 ESTABLISHED 状态，表明客户端这边已经准备好，但TCP 需要两端都准备好才可以进行数据传输。服务器端收到客户端的 ACK 之后会从 SYN_RCVD 状态转移到 ESTABLISHED 状态，表明服务器端也准备好进行数据传输了。这样客户端和服务器端都是 ESTABLISHED 状态，就可以进行后面的数据传输了。所以 ESTABLISHED 也可以说是一个数据传送状态。

    + 下面看看TCP四次挥手过程的状态变迁。

      1. **`FIN_WAIT_1:`** 第一次挥手。主动关闭的一方（执行主动关闭的一方既可以是客户端，也可以是服务器端，这里以客户端执行主动关闭为例），终止连接时，发送**`FIN`**给对方，然后等待对方返回**`ACK`**。调用**`close()`**第一次挥手就进入此状态。
      2. **CLOSE_WAIT**：接收到FIN 之后，被动关闭的一方进入此状态。具体动作是接收到 FIN，同时发送 ACK。之所以叫 CLOSE_WAIT 可以理解为被动关闭的一方此时正在等待上层应用程序发出关闭连接指令。TCP关闭是全双工过程，这里客户端执行了主动关闭，被动方服务器端接收到FIN 后也需要调用 close 关闭，这个 CLOSE_WAIT 就是处于这个状态，等待发送 FIN，发送了FIN 则进入 LAST_ACK 状态。
      3. **FIN_WAIT_2**：主动端（这里是客户端）先执行主动关闭发送FIN，然后接收到被动方返回的 ACK 后进入此状态。
      4. **LAST_ACK**：被动方（服务器端）发起关闭请求，由状态2 进入此状态，具体动作是发送 FIN给对方，同时在接收到ACK 时进入CLOSED状态。
      5. **CLOSING**：两边同时发起关闭请求时（即主动方发送FIN，等待被动方返回ACK，同时被动方也发送了FIN，主动方接收到了FIN之后，发送ACK给被动方），主动方会由FIN_WAIT_1 进入此状态，等待被动方返回ACK。
      6. **TIME_WAIT**：从状态变迁图会看到，四次挥手操作最后都会经过这样一个状态然后进入CLOSED状态。

    + | 状态            | 描述                                                   |
      | :-------------- | ------------------------------------------------------ |
      | **CLOSED**      | 阻塞或关闭状态，表示主机当前没有正在传输或者建立的链接 |
      | **LISTEN**      | 监听状态，表示服务器做好准备，等待建立传输链接         |
      | **SYN RECV**    | 收到第一次的传输请求，还未进行确认                     |
      | **SYN SENT**    | 发送完第一个SYN报文，等待收到确认                      |
      | **ESTABLISHED** | 链接正常建立之后进入数据传输阶段                       |
      | **FIN WAIT1**   | 主动发送第一个FIN报文之后进入该状态                    |
      | **FIN WAIT2**   | 已经收到第一个FIN的确认信号，等待对方发送关闭请求      |
      | **TIMED WAIT**  | 完成双向链接关闭，等待分组消失                         |
      | **CLOSING**     | 双方同时关闭请求，等待对方确认时                       |
      | **CLOSE WAIT**  | 收到对方的关闭请求并进行确认进入该状态                 |
      | **LAST ACK**    | 等待最后一次确认关闭的报文                             |

25. ### 简述 TCP 慢启动

    **慢启动**（Slow Start），是传输控制协议（TCP）使用的一种阻塞控制机制。慢启动也叫做指数增长期。慢启动是指每次TCP接收窗口收到确认时都会增长。增加的大小就是已确认段的数目。这种情况一直保持到要么没有收到一些段，要么窗口大小到达预先定义的阈值。如果发生丢失事件，TCP就认为这是网络阻塞，就会采取措施减轻网络拥挤。一旦发生丢失事件或者到达阈值，TCP就会进入线性增长阶段。这时，每经过一个RTT窗口增长一个段。

26. #### ==介绍一下数据库分页==

    + 在MySQL中，SELECT语句默认返回所有匹配的行，它们可能是指定表中的每个行。为了返回第一行或前几行，可使用LIMIT子句，以实现分页查询。LIMIT子句的语法如下：

      > ```mysql
      > -- 在所有的查询结果中，返回前5行记录。 SELECT prod_name FROM products LIMIT 5; -- 在所有的查询结果中，从第5行开始，返回5行记录。 SELECT prod_name FROM products LIMIT 5,5;
      > ```

    + 总之，带一个值的LIMIT总是从第一行开始，给出的数为返回的行数。带两个值的LIMIT可以指定从行号为第一个值的位置开始。

27. ==说说 TCP 如何保证有序==

    + 主机每次发送数据时，TCP就给每个数据包分配一个序列号并且在一个特定的时间内等待接收主机对分配的这个序列号进行确认，如果发送主机在一个特定时间内没有收到接收主机的确认，则发送主机会重传此数据包。接收主机利用序列号对接收的数据进行确认，以便检测对方发送的数据是否有丢失或者乱序等，接收主机一旦收到已经顺序化的数据，它就将这些数据按正确的顺序重组成数据流并传递到高层进行处理。

    + 具体步骤如下:

      > + 为了保证数据包的可靠传递，发送方必须把已发送的数据包保留在缓冲区；
      > + 并为每个已发送的数据包启动一个超时定时器；
      > + 如在定时器超时之前收到了对方发来的应答信息（可能是对本包的应答，也可以是对本包后续包的应答），则释放该数据包占用的缓冲区;
      > + 否则，重传该数据包，直到收到应答或重传次数超过规定的最大次数为止。
      > + 接收方收到数据包后，先进行CRC校验，如果正确则把数据交给上层协议，然后给发送方发送一个累计应答包，表明该数据已收到，如果接收方正好也有数据要发给发送方，应答包也可方在数据包中捎带过去。

28. ### ==简述 TCP 超时重传==

    TCP可靠性中最重要的一个机制是处理数据超时和重传。TCP协议要求在发送端每发送一个报文段，就启动一个定时器并等待确认信息；接收端成功接收新数据后返回确认信息。若在定时器超时前数据未能被确认，TCP就认为报文段中的数据已丢失或损坏，需要对报文段中的数据重新组织和重传。

29. ### ==说说 TCP 可靠性保证==

    + TCP主要提供了检验和、序列号/确认应答、超时重传、最大消息长度、滑动窗口控制等方法实现了可靠性传输。

    + **检验和**

      > 通过检验和的方式，接收端可以检测出来数据是否有差错和异常，假如有差错就会直接丢弃TCP段，重新发送。TCP在计算检验和时，会在TCP首部加上一个12字节的伪首部。检验和总共计算3部分：TCP首部、TCP数据、TCP伪首部

    + #### **序列号/确认应答**

      > + 这个机制类似于问答的形式。比如在课堂上老师会问你“明白了吗？”，假如你没有隔一段时间没有回应或者你说不明白，那么老师就会重新讲一遍。其实计算机的确认应答机制也是一样的，发送端发送信息给接收端，接收端会回应一个包，这个包就是应答包。
      > + 上述过程中，只要发送端有一个包传输，接收端没有回应确认包（ACK包），都会重发。或者接收端的应答包，发送端没有收到也会重发数据。这就可以保证数据的完整性。

    + #### **超时重传**

      > - 超时重传是指发送出去的数据包到接收到确认包之间的时间，如果超过了这个时间会被认为是丢包了，需要重传。那么我们该如何确认这个时间值呢？
      > - 我们知道，一来一回的时间总是差不多的，都会有一个类似于平均值的概念。比如发送一个包到接收端收到这个包一共是0.5s，然后接收端回发一个确认包给发送端也要0.5s，这样的两个时间就是RTT（往返时间）。然后可能由于网络原因的问题，时间会有偏差，称为抖动（方差）。
      > - 从上面的介绍来看，超时重传的时间大概是比往返时间+抖动值还要稍大的时间。
      > - 但是在重发的过程中，假如一个包经过多次的重发也没有收到对端的确认包，那么就会认为接收端异常，强制关闭连接。并且通知应用通信异常强行终止。

    + #### **最大消息长度**

      > + 在建立TCP连接的时候，双方约定一个最大的长度（MSS）作为发送的单位，重传的时候也是以这个单位来进行重传。理想的情况下是该长度的数据刚好不被网络层分块。

    + #### **滑动窗口控制**

      > + 我们上面提到的超时重传的机制存在效率低下的问题，发送一个包到发送下一个包要经过一段时间才可以。所以我们就想着能不能不用等待确认包就发送下一个数据包呢？这就提出了一个滑动窗口的概念。
      > + 窗口的大小就是在无需等待确认包的情况下，发送端还能发送的最大数据量。这个机制的实现就是使用了大量的缓冲区，通过对多个段进行确认应答的功能。通过下一次的确认包可以判断接收端是否已经接收到了数据，如果已经接收了就从缓冲区里面删除数据。
      > + 在窗口之外的数据就是还未发送的和对端已经收到的数据。那么发送端是怎么样判断接收端有没有接收到数据呢？或者怎么知道需要重发的数据有哪些呢？通过下面这个图就知道了。

    + #### **拥塞控制**

      > - 窗口控制解决了 两台主机之间因传送速率而可能引起的丢包问题，在一方面保证了TCP数据传送的可靠性。然而如果网络非常拥堵，此时再发送数据就会加重网络负担，那么发送的数据段很可能超过了最大生存时间也没有到达接收方，就会产生丢包问题。为此TCP引入慢启动机制，先发出少量数据，就像探路一样，先摸清当前的网络拥堵状态后，再决定按照多大的速度传送数据。
      > - 发送开始时定义拥塞窗口大小为1；每次收到一个ACK应答，拥塞窗口加1；而在每次发送数据时，发送窗口取拥塞窗口与接送段接收窗口最小者。
      > - 慢启动：在启动初期以指数增长方式增长；设置一个慢启动的阈值，当以指数增长达到阈值时就停止指数增长，按照线性增长方式增加至拥塞窗口；线性增长达到网络拥塞时立即把拥塞窗口置回1，进行新一轮的“慢启动”，同时新一轮的阈值变为原来的一半。

30. ### 简述 TCP 的 TIME_WAIT，为什么需要有这个状态

    1. TIME_WAIT状态也成为2MSL等待状态。每个具体TCP实现必须选择一个报文段最大生存时间MSL（Maximum Segment Lifetime），它是任何报文段被丢弃前在网络内的最长时间。这个时间是有限的，因为TCP报文段以IP数据报在网络内传输，而IP数据报则有限制其生存时间的TTL字段。

       对一个具体实现所给定的MSL值，处理的原则是：当TCP执行一个主动关闭，并发回最后一个ACK，该连接必须在TIME_WAIT状态停留的时间为2倍的MSL。这样可让TCP再次发送最后的ACK以防这个ACK丢失（另一端超时并重发最后的FIN）。

       这种2MSL等待的另一个结果是这个TCP连接在2MSL等待期间，定义这个连接的插口（客户的IP地址和端口号，服务器的IP地址和端口号）不能再被使用。这个连接只能在2MSL结束后才能再被使用。

    2. 理论上，四个报文都发送完毕，就可以直接进入CLOSE状态了，但是可能网络是不可靠的，有可能最后一个ACK丢失。所以TIME_WAIT状态就是用来重发可能丢失的ACK报文。

31. ### 简述什么是 MSL，为什么客户端连接要等待2MSL的时间才能完全关闭

    1. MSL是Maximum Segment Lifetime的英文缩写，可译为“最长报文段寿命”，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。
    2. 为了保证客户端发送的最后一个ACK报文段能够到达服务器。因为这个ACK有可能丢失，从而导致处在LAST-ACK状态的服务器收不到对FIN-ACK的确认报文。服务器会超时重传这个FIN-ACK，接着客户端再重传一次确认，重新启动时间等待计时器。最后客户端和服务器都能正常的关闭。假设客户端不等待2MSL，而是在发送完ACK之后直接释放关闭，一但这个ACK丢失的话，服务器就无法正常的进入关闭连接状态。

    - 两个理由：

      - 保证客户端发送的最后一个ACK报文段能够到达服务端。

        这个ACK报文段有可能丢失，使得处于LAST-ACK状态的B收不到对已发送的FIN+ACK报文段的确认，服务端超时重传FIN+ACK报文段，而客户端能在2MSL时间内收到这个重传的FIN+ACK报文段，接着客户端重传一次确认，重新启动2MSL计时器，最后客户端和服务端都进入到CLOSED状态，若客户端在TIME-WAIT状态不等待一段时间，而是发送完ACK报文段后立即释放连接，则无法收到服务端重传的FIN+ACK报文段，所以不会再发送一次确认报文段，则服务端无法正常进入到CLOSED状态。

      - 防止“已失效的连接请求报文段”出现在本连接中。

        客户端在发送完最后一个ACK报文段后，再经过2MSL，就可以使本连接持续的时间内所产生的所有报文段都从网络中消失，使下一个新的连接中不会出现这种旧的连接请求报文段。

32. ### 服务器怎么判断客户端断开了连接

    1. 检测连接是否丢失的方法大致有两种：**keepalive**和**heart-beat**
    2. （tcp内部机制）采用keepalive，它会先要求此连接一定时间没有活动（一般是几个小时），然后发出数据段，经过多次尝试后（每次尝试之间也有时间间隔），如果仍没有响应，则判断连接中断。可想而知，整个**周期需要很长**的时间。
    3. （应用层实现）一个简单的heart-beat实现一般测试连接是否中断采用的时间间隔都比较短，可以**很快的决定连接是否中断**。并且，由于是在应用层实现，因为可以自行决定当判断连接中断后应该采取的行为，而keepalive在判断连接失败后只会将连接丢弃。

33. ==说说你对`SQL`注入的理解==

    在浏览器的某个网页中，在身份验证框或者数据查询框中，通过输入sql命令，影响SQL语句字符串的拼接，从而使原本执行不通过的SQL语句可以执行通过。

34. ==说一说进程调度算法有哪些==

    - 先来先服务（FCFS）
    - 最短任务优先（SJF）
    - 最短完成时间优先（STCF）
    - 时间片轮转
    - 优先级调度
    - 多级反馈队列

35. ==`C++`的多态如何实现==

    + `C++`的多态性，一言以蔽之就是：

      > 在基类的函数前加上`virtual`关键字，在派生类中重写该函数，运行时将会根据所指对象的实际类型来调用相应的函数，如果对象类型是派生类，就调用派生类的函数，如果对象类型是基类，就调用基类的函数。

    + 虚表:虚函数表的缩写，类中含有virtual关键字修饰的方法时，编译器会自动生成虚表

    + 虚表指针:在含有虚函数的类实例化对象时，对象地址的前四个字节存储的指向虚表的指针

      ![](https://img-blog.csdn.net/20180415215722290)

      <img src="https://img-blog.csdn.net/20180415215659513"  />

      **上图中展示了虚表和虚表指针在基类对象和派生类对象中的模型，下面阐述实现多态的过程**

      (1) ==编译器在发现基类中有虚函数时，会自动为每个含有虚函数的类生成一份虚表，该表是一个一维数组，虚表里保存了虚函数的入口地址==

      (2) ==编译器会在每个对象的前四个字节中保存一个虚表指针，即vptr，指向对象所属类的虚表。在构造时，根据对象的类型去初始化虚指针vptr，从而让vptr指向正确的虚表，从而在调用虚函数时，能找到正确的函数==

      (3) ==所谓的合适时机，在派生类定义对象时，程序运行会自动调用构造函数，在构造函数中创建虚表 并对虚表初始化。在构造子类对象时，会先调用父类的构造函数，此时，编译器只“看到了”父类，并为父类对象初始化虚表指针，令它指向父类的虚表;当调用子类的构造函数时，为子类对象初始化虚表指 针，令它指向子类的虚表==

      (4) ==当派生类对基类的虚函数没有重写时，派生类的虚表指针指向的是基类的虚表;当派生类对基类 的虚函数重写时，派生类的虚表指针指向的是自身的虚表;当派生类中有自己的虚函数时，在自己的虚表中将此虚函数地址添加在后面==

      这样指向派生类的基类指针在运行时，就可以根据派生类对虚函数重写情况动态的进行调用，从而实现多态性。

36. ==`TCP/IP`协议栈层次及作用==

    + 应用层：==应用层是网络应用程序及它们的应用层协议存留的地方。==
    + 传输层：==运输层在应用程序端点之间传送应用层报文。==
    + 网络层：==网络层负责将网络层数据报从一台主机移动到另一台主机。==
    + 链路层：==在每个节点，网络层将数据报下传给链路层，链路层沿着路径将数据报传递给下一个节点。在该下一个节点，链路层将数据报上传给网络层。==
    + 物理层：==将该帧中的一个个比特从一个节点移动到下一个节点。==

37. ==`select/poll/epoll`区别==

    - 调用函数

    - - select和poll都是一个函数，epoll是一组函数

    - 文件描述符数量

    - - select通过线性表描述文件描述符集合，文件描述符有上限，一般是1024，但可以修改源码，重新编译内核，不推荐
      - poll是链表描述，突破了文件描述符上限，最大可以打开文件的数目
      - epoll通过红黑树描述，最大可以打开文件的数目，可以通过命令ulimit -n number修改，仅对当前终端有效

    - 将文件描述符从用户传给内核

    - - select和poll通过将所有文件描述符拷贝到内核态，每次调用都需要拷贝
      - epoll通过epoll_create建立一棵红黑树，通过epoll_ctl将要监听的文件描述符注册到红黑树上

    - 内核判断就绪的文件描述符

    - - select和poll通过遍历文件描述符集合，判断哪个文件描述符上有事件发生
      - epoll_create时，内核除了帮我们在epoll文件系统里建了个红黑树用于存储以后epoll_ctl传来的fd外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。
      - epoll是根据每个fd上面的回调函数(中断函数)判断，只有发生了事件的socket才会主动的去调用 callback函数，其他空闲状态socket则不会，若是就绪事件，插入list

    - 应用程序索引就绪文件描述符

    - - select/poll只返回发生了事件的文件描述符的个数，若知道是哪个发生了事件，同样需要遍历
      - epoll返回的发生了事件的个数和结构体数组，结构体包含socket的信息，因此直接处理返回的数组即可

    - 工作模式

    - - select和poll都只能工作在相对低效的LT模式下
      - epoll则可以工作在ET高效模式，并且epoll还支持EPOLLONESHOT事件，该事件能进一步减少可读、可写和异常事件被触发的次数。 

    - 应用场景

    - - 当所有的fd都是活跃连接，使用epoll，需要建立文件系统，红黑书和链表对于此来说，效率反而不高，不如selece和poll
      - 当监测的fd数目较小，且各个fd都比较活跃，建议使用select或者poll
      - 当监测的fd数目非常大，成千上万，且单位时间只有其中的一部分fd处于就绪状态，这个时候使用epoll能够明显提升性能

38. ==冒泡排序==

    ```C++
    #include <iostream>
    using namespace std;
    template <typename T>
    void bubble_sort (T arr[], int len) {
     	int i, j;
      for (i = 0; i < len - 1; i++)
        for (j = 0; j < len - 1 - i; j++)
          if (arr[j] > arr[j + 1])
            swap (arr[j], arr[j+1]);
    }
    
    int main ()
    {
      int arr[] = {61, 17, 29, 22, 34, 60, 72, 21, 50, 1, 62};
      int len = (int) sizeof (arr) / sizeof (*arr);
      bubble_sort (arr, len);
      for (auto elem : arr)
        	cout << elem << ' ';
      cout << endl;
      
      float arrf[] = {17.5. 19.1, 0.6, 1.9, 10.5, 12.4, 3.8, 19.7, 1.5, 25.4, 28.6,
                      4.4, 23.8, 5.4 };
      len = sizeof (arrf) / sizeof (*arrf);
      bubbule_sort (arrf, len);
      for (auto elem : arrf)
        cout << elem << ' ';
      cout << endl;
    }
    ```

39. ==选择排序==

    + 时间复杂度永远都是o(n^2^)。

    + ==唯一的好处可能就是不占用额外的内存空间==

    + 首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置。

    + 再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。

    + 重复第二步，直到所有元素均排序完毕。

      ```C++
      template <typename T>
      void selection_sort (std::vector<T>& arr) {
        for (int i = 0; i < arr.size() - 1; i++) {
          int min = i;
          for (int j = i + 1; j < arr.size(); j++)
            if (arr[j] < arr[min])
              min = j;
          std::swap (arr[i], arr[min]);
        }
      }
      ```

40. ==插入排序==

    + ==将第一待排序序列第一个元素看作一个有序序列，把第二个元素到最后一个元素当成是未排序序列。==
    + ==从头到尾依次扫描未排序序列，将扫描到的每个元素插入有序序列的适当位置。==

    ```C++
    void insertion_sort (int arr[], int len)
    {
      for (int i = 1; i < len; i++) {
        int key = arr[i];
        int j = i - 1;
        while ((j >= 0) && (key < arr[j])) {
          arr[j+1] = arr[j];
          j--;
        }
        arr[j+1] = key;
      }
    }
    ```

41. ==希尔排序==

    + 也称递减增量排序算法，是插入排序的一种更高效的改进版本。==但希尔排序是非稳定排序算法。==
    + ==希尔排序的思想是：==先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，待整个序列中的记录“基本有序”时，再对全体记录进行依次直接插入排序。

    ```C++
    template <typename T>
    void shell_sort (T arr[], int length)
    {
      int h = 1;
      while (h < length / 3) {
        h = 3 * h + 1;
      }
      while (h >= 1) {
        for (int i = h; i < length; ++i) {
          for (int j = i; j >= h && arr[j] < arr[j-h]; j -= h) {
            std::swap (array[j], array[j-h]);
          }
        }
        h = h / 3;
      }
    }
    ```

42. ==归并排序==

    + 归并排序是建立在归并操作上的一种有效的排序算法。==该算法是采用分治法的一个非常典型的应用。==

    + 作为一种典型的分而治之思想的算法应用，归并排序的实现有两种方法：

      + 自上而下的递归
      + 自下而上的迭代

    + ==始终都是`O(nlogn)`的时间复杂度，代价是需要额外的内存空间==

    + 算法步骤

      1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
      2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置
      3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
      4. 重复步骤3直到某一指针达到序列尾
      5. 将另一序列剩下的所有元素直接复制到合并序列尾

      ```C++
      template <typename T>
      void merge_sort (T arr[], int len)
      {
        T *a = arr;
        T *b = new T[len];
        for (int seg = 1; seg < len; seg += seg) {
          for (int start = 0; start < len; start += seg + seg) {
            int low = start, mid = min (start + seg, len), high = min (start + seg + seg, len);
            int k = low;
            int start1 = low, end1 = mid;
            int start2 = mid, end2 = high;
            while (start1 < endl1 && start2 < end2) 
              b[k++] = a[start1] < a[start2] ? a[start1++] : a[start2++];
            while (start1 < end1)
              b[k++] = a[start1++];
            while (start2 < end2)
              b[k++] = a[start2++];
          }
          T *temp = a;
          a = b;
          b = temp;
        }
        
        if (a != arr) {
          for (int i = 0; i < len; i++)
            b[i] = a[i];
          b = a;
        }
        delete [] b;
      }
      ```

      ```C++
      void Merge (vector<int> &Array, int front, int mid, int end) 
      {
        vector<int> LeftSubArray (Array.begin() + front, Array.begin() + min + 1);
        vector<int> RightSubArray (Array.beign() + mid + 1, Array.begin() + end + 1);
        int idxLeft = 0, idxRight = 0;
        LeftSubArray.insert (LeftSubArray.end(), numeric_limits<int>::max());
        RightSubArray.insert (RightSubArray.end(), numeric_limits<int>::max());
        for (int i = front; i <= end; i++) {
          if (LefSubArray[idxLeft] < RightSubArray[idxRight]) {
            Array[i] = LeftSubArray[idxLeft];
            idxLeft++;
          }
          else {
            Array[i] = RightSubArray[idxRight];
            idxRight++;
          }
        }
      }
      
      void MergeSort (vector<int> &Array, int front, int end)
      {
        if (front >= end)
          return;
        
        int mid = (front + end) / 2;
        MergeSort (Array, front, mid);
        MergeSort (Array, mid + 1, end);
        Merge (Array, front, mid, end);
      }
      ```

43. ==快速排序==

    + 在平均情况下，排序nn个项目要`O(nlogn)`次比较，在最坏情况下则需要`O(n^2^)`次比较，但这种情况并不常见。

    + ==快速排序使用分治法策略来把一个串行分为两个子串。

    + 从本质上来看，快速排序应该算是在冒泡排序基础上的递归分治法。

    + ==算法步骤==

      1. 从数列中挑出一个元素，成为“基准”
      2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区操作。
      3. 递归地把小于基准值元素的子数列和大于基准值元素的子数列排序

      ```C++
      void partition1 (int A[], int low, int high)
      {
        int pivot = A[low];
        while (low < high) {
          while (low < high && A[high] >= pivot) {
            --high;
          }
          A[low] = A[high];
          while (low < high && A[low] <= pivot) {
            ++low;
          }
          A[high] = A[low];
        }
        A[low] = pivot;
        return low;
      }
      
      void QuickSort (int A[], int low, int high) 
      {
        if (low < high) {
          int pivot = Partition1 (A, low, high);
          QuickSort (A, low, pivot - 1);
          QuickSort (A, pivot + 1, high);
        } 
      }
      ```

44. ==堆排序==

45. ==有限状态机==

    有限状态机是逻辑单元内部的一种高效编程方法，在服务器编程中，服务器可以根据不同状态或者消息类型进行相应的处理逻辑，使得程序逻辑清晰易懂。

46. 
