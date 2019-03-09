# Runtime源码 类、对象、isa
OC做为一门动态语言，runtime是其最大的特点，它是一套底层的 C 语言 API，是 iOS 系统的核心之一。开发者在编码过程中，可以给任意一个对象发送消息，在编译阶段只是确定了要向接收者发送这条消息，而接受者将要如何响应和处理这条消息，那就要看运行时来决定了。在日常开发过程中类、对象、属性算是最常见的了，今天以对象为切入点，分析一下对象和类在runtime层面的表示。
## 类
类在runtime的表示为：
<pre>
struct objc_object {
private:
    isa_t isa;
    ...
}
</pre>
最关键的就是**isa_t**这个类型和一系列的构造函数，点击查看**isa_t**发现这是一个联合体：
<pre>
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
 }
</pre>
可以看到这个联合体里面有个Class类型的属性cls，看起来里面应该是关于这个对象的类的相关信息，那我们再看看Class包含了哪些内容。
<pre>
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    ...
}
</pre>
由此可见Class是一个objcclass类型的结构体，而objcclas继承字objc_object，说明类也是一个对象，只是比普通的对象多了一些属性，比如superclass等。  
先不看这些属性，这里还有一个很奇怪的问题，既然类也是一个objc_object，那就是说类也有一个isa指针，那类的isa指针指向哪里呢？查看了不少资料，这篇讲的挺好:[Classes and metaclasses](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)  
大致意思是在class之上还有叫做元类（meta class）的存在，而class的isa指针就是指向对应的meta class。  

我们都知道class中存储的是描述对象的相关信息，那么相应的meta class中存放的就是描述class相关信息。说的更直白一点，在我们写代码时，通过对象来调用的方法（实例方法）都是存储在class中的，通过类名来调用的方法（类方法）都是存储在meta class中的。

到这里对象和类的关系已经比较清楚了，但是如果细细思考一下，会发现还有一个问题，就是meta class也是有isa指针的，那么这个isa又指向了哪里呢？在上面给出的那篇文章里面有这么一张图：
![class diagram](https://upload-images.jianshu.io/upload_images/4670835-0b5ed7e3cd00094a.jpeg)
由图可知：meta class的isa指向root meta class（绝大部分情况下是NSObject），root meta class的isa指针指向自己。
## isa_t
先看看完整的**isa_t**,因为运行环境是osx，所以只截取x86_64部分，arm64的区别只在于部分字段的位数不同，字段是完全相同的：

```
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }   
    Class cls;
    uintptr_t bits;  
#   __x86_64__  
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; 
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };
}
```
看这个定义只能大概看出个框架，下面从isa的初始化过程来看看isa_t究竟是如何存储类或者元类的相关信息。

```
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 	
{ 
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        isa_t newisa(0);
        newisa.bits = ISA_MAGIC_VALUE;
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
        isa = newisa;
    }
}

```
上来就看不懂，nonpointer是个什么，为什么在这里传的是true？在之前那位大神的另一篇文章中也有解释：[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)。  

大概的意思是在64位系统中，为了降低内存使用，提升性能，isa中有一部分字段用来存储其他信息。这也解释了上面isa_t的那部分结构体。

>这有点像taggedPointer，两者之间有什么区别？备注一下后面再研究。  

现在知道了nonpointer为什么是true，那么把initIsa方法先简化一下:

```
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    isa_t newisa(0);
    newisa.bits = ISA_MAGIC_VALUE;
    newisa.has_cxx_dtor = hasCxxDtor;
    newisa.shiftcls = (uintptr_t)cls >> 3;
    isa = newisa;
}
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
```

一共三部分：

1. newisa.bits = ISA_MAGIC_VALUE;  
从ISA_MAGIC_VALUE的定义中可以看到这个字段初始化了两个部分，一个是magic字段(6位:111011)，一个是nonpointer字段(1位:1)，magic字段用于校验，nonpointer之前已经详细分析过了。  

2. newisa.has_cxx_dtor = hasCxxDtor;
这个字段存储类是否有c++析构器。  

3. newisa.shiftcls = (uintptr_t)cls >> 3;
将cls右移3位存到shiftcls中，从isa_t的结构体中也可以看到低3位都是用来存储其他信息的，既然可以右移三位，那就代表类地址的低三位全部都是0，否则就出错了，补0的作用应该是为了字节对齐。  

因为nonpointer的缘故，isa并不只是用来存储类地址了，所以需要提供一个额外的方法来返回真正的地址：

```
inline Class 
objc_object::ISA() 
{
    return (Class)(isa.bits & ISA_MASK);
}

#   define ISA_MASK        0x00007ffffffffff8ULL
```
其实就是取isa_t结构体的shiftcls字段。

### 其他字段
还有一些其他的字段，把上面那篇文章中相关部分翻译过来放在下面：

> // 是否曾经或正在被关联引用，如果没有，可以快速释放内存
uintptr_t has_assoc : 1;

> // 对象是否曾经或正在被弱引用，如果没有，可以快速释放内存
uintptr_t weakly_referenced : 1;

> // 对象是否正在释放内存
uintptr_t deallocating : 1;

>// 对象的引用计数太大，无法存储
uintptr_t has_sidetable_rc : 1;  

>// 对象的引用计数超过1，比如10，则此值为9
uintptr_t extra_rc : 8;

资料：[对象、类和isa](https://www.jianshu.com/p/a8eade8a1c6d)  
[isa和class](https://www.jianshu.com/p/9d649ce6d0b8)











