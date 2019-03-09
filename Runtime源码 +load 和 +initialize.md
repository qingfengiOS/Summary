#Runtime源码 +load 和 +initialize
## 一、前言
在iOS的开发中，Runtime的方法交换都是写在```+load```之中，为什么不是```+initialize```中呢？可能不少朋友对此或多或少有一点点的疑问。 我们知道：OC中几乎所有的类都继承自NSObject，而```+load```和```+initialize```用于类的初始化，这两者的区别和联系到底何在呢？接下来我们一起来看看这二者的区别。

## 二、+load
根据[官方文档](https://developer.apple.com/documentation/objectivec/nsobject/1418815-load?language=objc)的描述:  

1.  ```+load```是当一个类或者分类被添加到Objective-C运行时调用的。  
2. 本类```+load```的调用在其所有的父类```+load```调用之后。
3. 分类的```+load```在类的调用之后。  

> 也就是说调用顺序：父类 > 类 > 分类  
> 但是不同类之间的+load方法的调用顺序是不确定的。  

由于```+load```是在类第一次加载进Runtime运行时才调用，由此我们可以知道：
  
- 它只会调用一次，这也是为什么么方法交换写在```+load```的原因。
- 它的调用时机在```main```函数之前。  

## 三、+load的实现  
在Runtime源码的```objc-runtime-new.mm```和```objc-runtime-old.mm```中的```load_images```方法中都存在这关键代码：

```
prepare_load_methods((const headerType *)mh);
```  
和  
```
call_load_methods();
```  
### 3.1 prepare_ load_methods  

```
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertWriting();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```  
这里就是准备好满足 +load 方法调用条件的类和分类，而对**class**和**category**分开做了处理。   

- 在处理**class**的时候，调用了```schedule_class_load```:  

```
/***********************************************************************
* prepare_load_methods
* Schedule +load for classes in this image, any un-+load-ed 
* superclasses in other images, and any categories in this image.
**********************************************************************/
// Recursively schedule +load for cls and any un-+load-ed superclasses.
// cls must already be connected.
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```
**从倒数第三句代码看出这里对参数class的父类进行了递归调用，以此确保父类的优先级**

然后调用了```add_class_to_loadable_list```,把class加到了**loadable_classes**中：

```
/***********************************************************************
* add_class_to_loadable_list
* Class cls has just become connected. Schedule it for +load if
* it implements a +load method.
**********************************************************************/
void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```

- 对于分类则是调用```add_category_to_loadable_list```把category加入到**loadable_categories**之中：

```
/***********************************************************************
* add_category_to_loadable_list
* Category cat's parent class exists and the category has been attached
* to its class. Schedule this category for +load after its parent class
* becomes connected and has its own +load method called.
**********************************************************************/
void add_category_to_loadable_list(Category cat)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = _category_getLoadMethod(cat);

    // Don't bother if cat has no +load method
    if (!method) return;

    if (PrintLoading) {
        _objc_inform("LOAD: category '%s(%s)' scheduled for +load", 
                     _category_getClassName(cat), _category_getName(cat));
    }
    
    if (loadable_categories_used == loadable_categories_allocated) {
        loadable_categories_allocated = loadable_categories_allocated*2 + 16;
        loadable_categories = (struct loadable_category *)
            realloc(loadable_categories,
                              loadable_categories_allocated *
                              sizeof(struct loadable_category));
    }

    loadable_categories[loadable_categories_used].cat = cat;
    loadable_categories[loadable_categories_used].method = method;
    loadable_categories_used++;
}
```
### 3.2 call_ load_methods  

```
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```
从注释我们可以明确地看出：   

1. 重复地调用class里面的```+load ```方法  
2. 一次调用category里面的```+load```方法

还是看一下调用具体实现，以```call_class_loads ```为例:  

```
/***********************************************************************
* call_class_loads
* Call all pending class +load methods.
* If new classes become loadable, +load is NOT called for them.
*
* Called only by call_load_methods().
**********************************************************************/
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}
```
这就是真正负责调用类的 +load 方法了。它从上一步获取到的全局变量 **loadable_classes** 中取出所有可供调用的类，并进行清零操作。  

- ```loadable_classes``` 指向用于保存类信息的内存的首地址，  
- ```loadable_classes_allocated``` 标识已分配的内存空间大小，  
- ```loadable_classes_used``` 则标识已使用的内存空间大小。

然后，循环调用所有类的 +load 方法。注意，这里是（调用分类的 +load 方法也是如此）直接使用函数内存地址的方式``` (*load_method)(cls, SEL_load)```; 对 +load 方法进行调用的，而不是使用发送消息 objc_msgSend 的方式。  

这样的调用方式就使得 +load 方法拥有了一个非常有趣的特性，那就是子类、父类和分类中的 +load 方法的实现是被区别对待的。也就是说如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的。同理，当一个类和它的分类都实现了 +load 方法时，两个方法都会被调用。因此，我们常常可以利用这个特性做一些“有趣”的事情，比如说方法交换[method-swizzling](https://nshipster.com/method-swizzling/)。  
## 四、+initialize  
根据官方文档[initialize](https://developer.apple.com/documentation/objectivec/nsobject/1418639-initialize?language=objc)的描述：  

1. Runtime发送```+initialize```消息是在类或者其子类第一次收到消息时，而且父类会在类之前接收到消息  
2. ```+initialize```的实现是线程安全的，多线程下会有线程等待   
3. 父类的```+initialize ```可能会被调用多次  

也就是说 +initialize 方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的 +initialize 方法是永远不会被调用的。那这样设计有什么好处呢？好处是显而易见的，那就是节省系统资源，避免浪费。

## 五、+initialize的实现  
在runtime源码```objc-runtime-new.mm```的方法```lookUpImpOrForward```中有如下代码片段：   

```
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    ···
    if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlockRead();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.read();
        
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }
}

```  
可以知道：在方法调用过程中，如果类没有被初始化的时候，会调用```_class_initialize```对类进行初始化，方法细节如下：   

```
/***********************************************************************
* class_initialize.  Send the '+initialize' message on demand to any
* uninitialized class. Force initialization of superclasses first.
**********************************************************************/
void _class_initialize(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
   * supercls = cls->superclass;
   * if (supercls  &&  !supercls->isInitialized()) {
   *     _class_initialize(supercls);
   * }
    
    // Try to atomically set CLS_INITIALIZING.
    {
        monitor_locker_t lock(classInitLock);
        if (!cls->isInitialized() && !cls->isInitializing()) {
            cls->setInitializing();
            reallyInitialize = YES;
        }
    }
    
    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.
        
        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);

        if (MultithreadedForkChild) {
            // LOL JK we don't really call +initialize methods after fork().
            performForkChildInitialize(cls, supercls);
            return;
        }
        
        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if 
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: thread %p: calling +[%s initialize]",
                         pthread_self(), cls->nameForLogging());
        }

        // Exceptions: A +initialize call that throws an exception 
        // is deemed to be a complete and successful +initialize.
        //
        // Only __OBJC2__ adds these handlers. !__OBJC2__ has a
        // bootstrapping problem of this versus CF's call to
        // objc_exception_set_functions().
        
        // Exceptions: A +initialize call that throws an exception 
        // is deemed to be a complete and successful +initialize.
        //
        // Only __OBJC2__ adds these handlers. !__OBJC2__ has a
        // bootstrapping problem of this versus CF's call to
        // objc_exception_set_functions().
#if __OBJC2__
        @try
#endif
        {
            callInitialize(cls);

            if (PrintInitializing) {
                _objc_inform("INITIALIZE: thread %p: finished +[%s initialize]",
                             pthread_self(), cls->nameForLogging());
            }
        }
		...
}
```
这源码我们可以可出结论：   

1. 从前面*的行数知道，_class_initialize方法会对class的父类进行递归调用，由此可以确保父类优先于子类初始化。
2. 在截出的代码末尾有着如下方法：```callInitialize(cls);```  

```
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
    asm("");
}
```
这里指明了```+initialize```的调用方式是objc_msgSend,它和普通方法一样是由Runtime通过发消息的形式，调用走的都是发送消息的流程。换言之，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 +initialize 方法，那么就会对这个类中的实现造成覆盖。

因此，如果一个子类没有实现 +initialize 方法，那么父类的实现是会被执行多次的。有时候，这可能是你想要的；但如果我们想确保自己的 +initialize 方法只执行一次，避免多次执行可能带来的副作用时，我们可以使用下面的代码来实现：  

```
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

## 总结
|  | +load | +initialize |
| ------ | :------: | :------: |
| 调用时机 | 加载到runtime时 | 收到第一条消息时，可能永远不调用 |
| 调用方式（本质） | 函数调用 | runtime调度（和普通的方法一样，通过objc_msgSend） |
| 调用顺序 | 父类 > 类 > 分类 | 父类 > 类 |
| 调用次数 | 一次 | 不定，可能多次可能不调用 |
| 是否沿用父类的实现 | 否 | 是 |
| 分类的中实现 | 类和分类都执行 | "覆盖"类中的方法，只执行分类的实现 | 
## 后记 
应该尽可能减少initialize的调用，节省资源，截取官方原文的供大家参考：  
>Because initialize is called in a blocking manner, it’s important to limit method implementations to the minimum amount of work necessary possible. Specifically, any code that takes locks that might be required by other classes in their initialize methods is liable to lead to deadlocks. Therefore, you should not rely on initialize for complex initialization, and should instead limit it to straightforward, class local initialization.  

更多资料：  
[load](https://developer.apple.com/documentation/objectivec/nsobject/1418815-load?language=objc)  
[initialize](https://developer.apple.com/documentation/objectivec/nsobject/1418639-initialize)  
[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)
