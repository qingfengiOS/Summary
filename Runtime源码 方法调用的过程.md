# Runtime源码 方法调用的过程
## 前言
Objective-C语言的一大特性就是动态的，根据[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW1)的描述：在runtime之前，消息和方法并不是绑定在一起的，编译器会把方法调用转换为```objc_msgSend(receiver, selector)```，如果方法中带有参数则转换为```objc_msgSend(receiver, selector, arg1, arg2, ...)```接下来我们通过源码一窥究竟，在次之前我们先了解几个基本概念  

- SEL
在objc.h文件中我们可以看到如下代码：  

```
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```
SEL其实就是一个不透明的类型它代表一个方法选择子，在编译期，会根据方法名字生成一个ID。

- IMP 
在objc.h文件中我们可以看到IMP：

```
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
#endif
```
他是一个函数指针，指向方法实现的首地址。  

- Method  

```
/// An opaque type that represents a method in a class definition.
typedef struct objc_method *Method;  

struct objc_method {
    SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
    char * _Nullable method_types                            OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```  
它保存了SEL到IMP和方法类型，所以我们可以通过SEL调用对应的IMP  

## 方法调用的流程  
objc_msgSend的消息分发分为以下几个步骤：
我们找到objc _msgSend源码，都是汇编，不过注释比较详尽

```
/********************************************************************
 *
 * id objc_msgSend(id self, SEL	_cmd,...);
 * IMP objc_msgLookup(id self, SEL _cmd, ...);
 *
 * objc_msgLookup ABI:
 * IMP returned in r11
 * Forwarding returned in Z flag
 * r10 reserved for our use but not used
 *
 ********************************************************************/
	
	.data
	.align 3
	.globl _objc_debug_taggedpointer_classes
_objc_debug_taggedpointer_classes:
	.fill 16, 8, 0
	.globl _objc_debug_taggedpointer_ext_classes
_objc_debug_taggedpointer_ext_classes:
	.fill 256, 8, 0

	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame
	MESSENGER_START

	NilTest	NORMAL

	GetIsaFast NORMAL		// r10 = self->isa
	CacheLookup NORMAL, CALL	// calls IMP on success

	NilTestReturnZero NORMAL

	GetIsaSupport NORMAL

// cache miss: go search the method lists
LCacheMiss:
	// isa still in r10
	MESSENGER_END_SLOW
	jmp	__objc_msgSend_uncached

	END_ENTRY _objc_msgSend

	
	ENTRY _objc_msgLookup

	NilTest	NORMAL

	GetIsaFast NORMAL		// r10 = self->isa
	CacheLookup NORMAL, LOOKUP	// returns IMP on success

	NilTestReturnIMP NORMAL

	GetIsaSupport NORMAL

// cache miss: go search the method lists
LCacheMiss:
	// isa still in r10
	jmp	__objc_msgLookup_uncached

	END_ENTRY _objc_msgLookup

	
	ENTRY _objc_msgSend_fixup
	int3
	END_ENTRY _objc_msgSend_fixup

	
	STATIC_ENTRY _objc_msgSend_fixedup
	// Load _cmd from the message_ref
	movq	8(%a2), %a2
	jmp	_objc_msgSend
	END_ENTRY _objc_msgSend_fixedup

```

就此我们大概可以了解到其调用流程：  

- 判断receiver是否为nil，也就是objc_msgSend的第一个参数self，也就是要调用的那个方法所属对象
- 从缓存里寻找，找到了则分发，否则
- 利用objc-class.mm中_ class _lookupMethodAndLoadCache3方法去寻找selector

	- 如果支持GC，忽略掉非GC环境的方法（retain等）
	- 从本class的method list寻找selector，如果找到，填充到缓存中，并返回selector，否则
	- 寻找父类的method list，并依次往上寻找，直到找到selector，填充到缓存中，并返回selector，否则
	- 调用_class_resolveMethod，如果可以动态resolve为一个selector，不缓存，方法返回，否则
 	- 转发这个selector，否则
	- 报错，抛出异常    
	
这里的**_ class _lookupMethodAndLoadCache3**其实就是对**lookUpImpOrForward**方法的调用：  

```
/***********************************************************************
* _class_lookupMethodAndLoadCache.
* Method lookup for dispatchers ONLY. OTHER CODE SHOULD USE lookUpImp().
* This lookup avoids optimistic cache scan because the dispatcher 
* already tried that.
**********************************************************************/
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{        
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```
对第五个参数**cache**传值为NO，因为在此之前已经做了一个查找这里**CacheLookup NORMAL, CALL**，这里是对缓存查找的一个优化。

接下来看一下lookUpImpOrForward的一些关键实现细节  

- 缓存查找优化

```
 // Optimistic cache lookup
 if (cache)  
     methodPC = _cache_getImp(cls, sel);
     if (methodPC) return methodPC;    
 }
```
这里有个判断，是否需要缓存查找，如果cache为NO则进入下一步  

-  检查被释放类  

```
// Check for freed class
if (cls == _class_getFreedObjectClass())
	return (IMP) _freedHandler;
``` 

**_class _getFreedObjectClass**的实现：  

```
/***********************************************************************
* _class_getFreedObjectClass.  Return a pointer to the dummy freed
* object class.  Freed objects get their isa pointers replaced with
* a pointer to the freedObjectClass, so that we can catch usages of
* the freed object.
**********************************************************************/
static Class _class_getFreedObjectClass(void)
{
    return (Class)freedObjectClass;
}
```  
注释写到，这里返回的被释放对象的指针，不是太理解，备注这以后再看看  

- 懒加载+initialize

```
// Check for +initialize
    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }
```  
在方法调用过程中，如果类没有被初始化的时候，会调用_class_initialize对类进行初始化，关于+initialize可以看之前的[Runtime源码 +load 和 +initialize](https://www.jianshu.com/p/cea0ace15da8)。  

- 加锁保证原子性  

```
	// The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    // be added but ignored indefinitely because the cache was re-filled 
    // with the old value after the cache flush on behalf of the category.
retry:
    methodListLock.lock();

    // Try this class's cache.

    methodPC = _cache_getImp(cls, sel);
    if (methodPC) goto done;
```  
这里又做了一次缓存查找，因为上一步执行了+initialize
>加锁这一部分只有一行简单的代码，其主要目的保证方法查找以及缓存填充（cache-fill）的原子性，保证在运行以下代码时不会有新方法添加导致缓存被冲洗（flush）。   

- 本类的方法列表查找  

```
// Try this class's method lists.

meth = _class_getMethodNoSuper_nolock(cls, sel);
if (meth) {
log_and_fill_cache(cls, cls, meth, sel);
 methodPC = method_getImplementation(meth);
 goto done;
}
``` 
这里调用了**log_ and_ fill_cache**这个后面来看，接下里就是  

- 父类方法列表查找

```
// Try superclass caches and method lists.

    curClass = cls;
    while ((curClass = curClass->superclass)) {
        // Superclass cache.
        meth = _cache_getMethod(curClass, sel, _objc_msgForward_impcache);
        if (meth) {
            if (meth != (Method)1) {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, curClass, meth, sel);
                methodPC = method_getImplementation(meth);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = _class_getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, curClass, meth, sel);
            methodPC = method_getImplementation(meth);
            goto done;
        }
    }
```
关于消息在列表方法查找的过程，根据官方文档如下：

![messaging1](https://upload-images.jianshu.io/upload_images/2598795-0d45c94f8f83fd00.gif?imageMogr2/auto-orient/strip)

这里沿着集成体系对父类的方法列表进行查找，找到了就调用**log_ and_ fill_cache**

log_ and_ fill_cach的实现：  
记录:

```
/***********************************************************************
* log_and_fill_cache
* Log this method call. If the logger permits it, fill the method cache.
* cls is the method whose cache should be filled. 
* implementer is the class that owns the implementation in question.
**********************************************************************/
static void
log_and_fill_cache(Class cls, Class implementer, Method meth, SEL sel)
{
#if SUPPORT_MESSAGE_LOGGING
    if (objcMsgLogEnabled) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    _cache_fill (cls, meth, sel);
}
```

内部调用了**_cache _fill**，填充缓存：  

```
/***********************************************************************
* _cache_fill.  Add the specified method to the specified class' cache.
* Returns NO if the cache entry wasn't added: cache was busy, 
*  class is still being initialized, new entry is a duplicate.
*
* Called only from _class_lookupMethodAndLoadCache and
* class_respondsToMethod and _cache_addForwardEntry.
*
* Cache locks: cacheUpdateLock must not be held.
**********************************************************************/
bool _cache_fill(Class cls, Method smt, SEL sel)
{
    uintptr_t newOccupied;
    uintptr_t index;
    cache_entry **buckets;
    cache_entry *entry;
    Cache cache;

    cacheUpdateLock.assertUnlocked();

    // Never cache before +initialize is done
    if (!cls->isInitialized()) {
        return NO;
    }

    // Keep tally of cache additions
    totalCacheFills += 1;

    mutex_locker_t lock(cacheUpdateLock);

    entry = (cache_entry *)smt;

    cache = cls->cache;

    // Make sure the entry wasn't added to the cache by some other thread 
    // before we grabbed the cacheUpdateLock.
    // Don't use _cache_getMethod() because _cache_getMethod() doesn't 
    // return forward:: entries.
    if (_cache_getImp(cls, sel)) {
        return NO; // entry is already cached, didn't add new one
    }

    // Use the cache as-is if it is less than 3/4 full
    newOccupied = cache->occupied + 1;
    if ((newOccupied * 4) <= (cache->mask + 1) * 3) {
        // Cache is less than 3/4 full.
        cache->occupied = (unsigned int)newOccupied;
    } else {
        // Cache is too full. Expand it.
        cache = _cache_expand (cls);

        // Account for the addition
        cache->occupied += 1;
    }

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot because the 
    // minimum size is 4 and we resized at 3/4 full.
    buckets = (cache_entry **)cache->buckets;
    for (index = CACHE_HASH(sel, cache->mask); 
         buckets[index] != NULL; 
         index = (index+1) & cache->mask)
    {
        // empty
    }
    buckets[index] = entry;

    return YES; // successfully added new cache entry
}
```  

这里还没找到实现则进入下一步，动态方法解析和消息转发，关于消息转发的细节我们下篇再看。  
 
## 方法缓存  
在上面截出的源码中我们多次看到了**cache**,下面我们就来看看这个，在```runtime.h```和```objc-runtime-new```cache的定义如下

```
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method _Nullable buckets[1]                              OBJC2_UNAVAILABLE;
};
```
```
struct cache_t {
  struct bucket_t *_buckets;
  mask_t _mask;
  mask_t _occupied;
  ...
}
```
这就是cache在runtime层面的表示，里面的字段和代表的含义类似 

- buckets  
数组表示的hash表，每个元素代表一个方法缓存  
- mask   
当前能达到的最大index（从0开始），，所以缓存的size（total）是mask+1  
- occupied  
被占用的槽位，因为缓存是以散列表的形式存在的，所以会有空槽，而occupied表示当前被占用的数目

而在_ buckets中包含了一个个的**cache_entry**和**bucket_t**（objc2.0的变更）：  

```
typedef struct {
    SEL name;     // same layout as struct old_method
    void *unused;
    IMP imp;  // same layout as struct old_method
} cache_entry;
```
cache_entry定义也包含了三个字段，分别是： 

- name，被缓存的方法名字
- unused，保留字段，还没被使用。
- imp，方法实现

```
struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;
    ...
}
``` 
而bucket_t则没有了老的unused，包含了两个字段：    

- key，方法的标志(和之前的name对应)
- imp, 方法的实现

## 后记  
从runtime的源码我们知道了方法调用的流程和方法缓存，有些附带的问题答案也就呼之欲出了：
  
- 方法缓存在元类的上，由第一节([Runtime源码 类、对象、isa](https://www.jianshu.com/p/174a94704aa0))我们就知道在objc_class的isa指向了他的元类，所以每个类都只有一份方法缓存，而不是每一个类的object都保存一份。  
- 在方法调用的父类方法列表查找过程中，如果命中了也会调用```_cache_fill (cls, meth, sel);```，所以即便是从父类取到的方法，也会存在类本身的方法缓存里。而当用一个父类对象去调用那个方法的时候，也会在父类的metaclass里缓存一份。
- 缓存容量限制，在上面的代码中我们注意到这个判断： 

```
// Use the cache as-is if it is less than 3/4 full
mask_t newOccupied = cache->occupied() + 1;
mask_t capacity = cache->capacity();
if (cache->isConstantEmptyCache()) {
    // Cache is read-only. Replace it.
    cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
}
else if (newOccupied <= capacity / 4 * 3) {
     // Cache is less than 3/4 full. Use it as-is.
}
else {
     // Cache is too full. Expand it.
     cache->expand();
}
```  
当cache为空时创建；当新的被占用槽数小于等于其容量的3/4时，直接使用；否则调用```cache->expand();```扩充容量：  

```
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    uint32_t oldCapacity = capacity();
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);
}
```  

- 为什么类的方法列表不直接做成散列表呢，做成list，还要单独缓存，多费事？  
散列表是没有顺序的，Objective-C的方法列表是一个list，是有顺序的
这个问题么，我觉得有以下三个原因：  
	- Objective-C在查找方法的时候会顺着list依次寻找，并且category的方法在原始方法list的前面，需要先被找到，如果直接用hash存方法，方法的顺序就没法保证。
	- list的方法还保存了除了selector和imp之外其他很多属性。
	- 散列表是有空槽的，会浪费空间。

相关资料：  
美团酒旅博文：[深入理解Objective-C：方法缓存](https://tech.meituan.com/DiveIntoMethodCache.html)  
官方文档：[Messaging](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW1)

