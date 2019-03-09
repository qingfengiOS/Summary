#Runtime源码 Category(分类)

## 一、概述
**Category**又叫分类，类别，类目，作为Objective-C 2.0之后添加的语言特性，Category在如今的OC工程中随处可见，它可以在即使不知道源码的情况下为类添加方法，根据官方文档[Category](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Category.html#//apple_ref/doc/uid/TP40008195-CH5-SW2)其主要作用有：  

1. 分离类的实现到独立的文件，这样做的好处有：
	- 减小单个文件的代码量（维护一个2000代码的类和维护四个500的代码的类差别还是比较明显的）。
	- 把不同功能组织到不同的Category。
	- 方便多人维护一个类
	- 按需加载想要的Category
2. 定义私有方法
3. 模拟多继承
4. 把framework的私有方法公开

>注：Category 有一个非常容易误用的场景，那就是用 Category 来覆写父类或主类的方法。虽然目前 Objective-C 是允许这么做的，但是这种使用场景是非常不推荐的。[使用 Category 来覆写方法](https://stackoverflow.com/questions/5272451/overriding-methods-using-categories-in-objective-c)有很多缺点，比如不能覆写 Category 中的方法、无法调用主类中的原始实现等，且很容易造成无法预估的行为。

## 二、底层实现
在objc4-723中Category的定义如下：

```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```  
- name   
分类名
- cls  
类
- instanceMethods  
实例方法列表
- classMethod  
类方法列表  
- protocols  
遵守的协议列表  
- instanceProperties  
实例属性列表
- _classProperties  
类属性列表  

从Category的结构可见：  
1. 它可以添加实例方法，类方法，甚至可以实现协议，添加属性(但是这里的属性不会自动生成实例变量和对应的set、get方法需要通过[关联对象](https://www.jianshu.com/p/c741ffe47cf1)实现)  
2. 不可以添加实例变量。

## 三、Category加载
对于OC运行时，在入口方法中： 

```
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```
category被附加到类上面是在map_images的时候发生的，在new-ABI的标准下，_objc_init里面的调用的map_images最终会调用objc-runtime-new.mm里面的_read_images方法，而在_read_images方法的结尾，有以下的代码片段：

```
    // Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }
   
```
这里做的就是： 

1. 把category的实例方法、协议以及属性添加到类上
2. 把category的类方法和协议添加到类的metaclass上

接着往里看，category的各种列表是怎么最终添加到类上的，就拿实例方法列表来说吧：
在上述的代码片段里，addUnattachedCategoryForClass只是把类和category做一个关联映射，而remethodizeClass才是真正去处理添加事宜的功臣。  

```
/***********************************************************************
* remethodizeClass
* Attach outstanding categories to an existing class.
* Fixes up cls's method list, protocol list, and property list.
* Updates method caches for cls and its subclasses.
* Locking: runtimeLock must be held by the caller
**********************************************************************/
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertWriting();

    isMeta = cls->isMetaClass();

    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}

```
可以看到：在remethodizeClass内部核心方法是：**attachCategories**，它才是真正地添加方法  

```
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

需要注意的有两点：  

1. category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA。
2. category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休^_^，殊不知后面可能还有一样名字的方法。

## 四、Category和Extension
- Extension
	- 在编译器决议，是类的一部分，在编译器和头文件的@interface和实现文件里的@implement一起形成了一个完整的类。
	- 伴随着类的产生而产生，也随着类的消失而消失。
	Extension一般用来隐藏类的私有消息，你必须有一个类的源码才能添加一个类的Extension，所以对于系统一些类，如NSString，就无法添加类扩展
- Category
	- 是运行期决议的
	- 类扩展可以添加实例变量，分类不能添加实例变量  
	 
	>因为在运行期，对象的内存布局已经确定，如果添加实例变量会破坏类的内部布局，这对编译性语言是灾难性的。 	


更多资料：  
[Category](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Category.html#//apple_ref/doc/uid/TP40008195-CH5-SW2)  
[深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)  
[Objective-C Category 的实现原理](http://blog.leichunfeng.com/blog/2015/05/18/objective-c-category-implementation-principle/)  
[iOS Category详解](http://www.imlifengfeng.com/blog/?p=474)
