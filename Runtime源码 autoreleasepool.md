#Runtime源码 autoreleasepool

## 前言
  
在iOS开发中，由于ARC的普遍使用，内存管理的问题好像不那么常见了，但了解Objective-C的内存管理机制依然是非常必要的，今天我们来看看autorelease的一些细节，在ARC时代几乎很少看到autorelease的身影了，唯一常见的应该就是在```main```函数中了：  

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```  
这里可以看到整个 iOS 的应用都是包含在一个自动释放池 block 中的。那么这个autoreleasepool到底是什么呢？接下来我们来一窥究竟。

## 结构
使用```clang -rewrite-objc main.m```将main函数所在的文件，转化为C++代码：

```
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```
可以看到```autoreleasepool```其实是类型为```__AtAutoreleasePool ```的结构体，那么它又是什么呢？我们来看看他的定义：  

```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
}
```
由此，我们可以知道：当使用autoreleasepool时，实际上是：

```
/* @autoreleasepool */ {
    void *atautoreleasepoolobj = objc_autoreleasePoolPush();
    ...// 自己的代码，接收到 autorelease 消息的对象会被添加到这个 autoreleasepool中
    objc_autoreleasePoolPop(atautoreleasepoolobj);
}
```

它其实是调用了```objc_autoreleasePoolPush```,这又是什么？怎么越看越多？别急，马上到了，继续吧，打开runtime源码工程，在```NSObject.mm```文件中我们看到，其实他是：  

```
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```  
最后，再来看看```AutoreleasePoolPage```:  

```
class AutoreleasePoolPage {
	...
	magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
    ...
}
```  
终于看到他的结构了，总的来说，其实每一个自动释放池都是由一系列的 AutoreleasePoolPage 组成的，并且每一个 AutoreleasePoolPage 的大小都是 4096 字节（16 进制 0x1000）根据注释``` The stack is divided into a doubly-linked list of pages. Pages are added ```可以知道，他是以双链表的形式存储的，也对应了其结构中的parent和child，接下里我们看看他其他字段：  

- magic  
magic是```magic_t```类型的结构体，它用来校验 AutoreleasePoolPage结构的完整性 
- *next  
指向最新添加的 autoreleased 对象的下一个位置，初始化时指向 begin() 
- thread  
当前线程
- parent  
指向父结点，第一个结点的 parent 值为 nil ；
- child   
指向子结点，最后一个结点的 child 值为 nil ；
- depth   
代表深度，从 0 开始，往后递增 1；
- hiwat  
代表 high water mark 。


### objc_autoreleasePoolPush()
上面我们看到了```objc_autoreleasePoolPush()```接下来，看看push的具体操作:  

```
static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }
```
这里有一个debug环境的判断，我们忽略，直接看到```autoreleaseFast()```:  

```
static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {
            return page->add(obj);
        } else if (page) {
            return autoreleaseFullPage(obj, page);
        } else {
            return autoreleaseNoPage(obj);
        }
    }
```
- 当前 page 存在且没有满时:  
直接将对象添加到当前 page 中，即 next 指向的位置；
- 当前 page 存在且已满时:  
创建一个新的 page ，并将对象添加到新创建的 page 中；
- 当前 page 不存在时即还没有 page 时:  
创建第一个 page ，并将对象添加到新创建的 page 中。  

每调用一次 push 操作就会创建一个新的 autoreleasepool ，即往 AutoreleasePoolPage 中插入一个POOL_BOUNDARY，并且返回插入的 POOL_BOUNDARY的内存地址。
>从定义``` #   define POOL_BOUNDARY nil```可知：这里的POOL_BOUNDARY其实就是nil的别名
 
### autorelease  
接下来我们看看autorelease的实现，追溯源码发现它的函数调用栈很深，依次为：  

1. ```-(id) autorelease```  
2. ```_objc_rootAutorelease(id obj)``` 
3. ```objc_object::rootAutorelease()``` 
4. ```objc_object::rootAutorelease2()``` 
5.  ```static inline id autorelease(id obj)``` 
6. ```static inline id *autoreleaseFast(id obj)```  

它以obj为参数，通过层层调用，最终还是调用到了```autoreleaseFast```方法，  
也就是说：它和 push 操作的实现基本一致，只是不过 push 操作插入的是一个 POOL_BOUNDARY ，而 autorelease 操作插入的是一个具体的 autoreleased 对象。
 
### objc_autoreleasePoolPop()  
他其实是调用了：  

```
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```
pop调用到了```releaseUntil```里面有这样的操作：  

```
while (this->next != stop) {
            // Restart from hotPage() every time, in case -release 
            // autoreleased more objects
            AutoreleasePoolPage *page = hotPage();

            // fixme I think this `while` can be `if`, but I can't prove it
            while (page->empty()) {
                page = page->parent;
                setHotPage(page);
            }

            page->unprotect();
            id obj = *--page->next;
            memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
            page->protect();

            if (obj != POOL_BOUNDARY) {
                objc_release(obj);
            }
        }
```

pop可以说是：以上面说到的POOL_BOUNDARY的内存地址为入参，根据传入的POOL_BOUNDARY，找到这个push所在的位置，对之前入栈的每个antorealse对象都发送一次- release消息，并移动next指针，当指针的地址移动到POOL_BOUNDARY的位置时，这个自动释放池内的对象释放完毕。  

## 应用
1. 当使用循环时，使用@autoreleasePool能降低内存峰值（详见《Effective Objective-C 2.0》第34条）
2. 遍历时尽量使用系统的enumerateObjectsUsingBlock，因为其内部有自动释放池，而for活着for-in没有

## 总结
- 自动释放池是由 AutoreleasePoolPage 以双向链表的方式实现的。
- 调用AutoreleasePoolPage::push向栈插入一个nil作为标记，并返回这个地址。
- 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中（由于后加，所以他们一定在nil所在位置之上）。
- 调用 AutoreleasePoolPage::pop 方法，传递push函数得到的地址，依次向栈中的对象发送 release 消息，并下移指针，直到push的nil所在位置，即完成了自动释放。

参考资料：  
[NSAutoreleasePool](https://developer.apple.com/documentation/foundation/nsautoreleasepool#//apple_ref/occ/cl/NSAutoreleasePool)  
[黑幕背后的Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)  
[Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)
