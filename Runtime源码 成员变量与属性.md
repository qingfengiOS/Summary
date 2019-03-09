#Runtime源码 成员变量与属性
上篇文章我们了解了[类、对象和isa](https://www.jianshu.com/p/174a94704aa0)在runtime中的表示，现在来看看runtime对成员变量和属性的处理。在此之前我们先看看一个重要的概念：**类型编码**

### 类型编码
编译器将每个方法的返回值和参数类型编码为一个字符串，并且将其与方法的selector关联在一起。这个编码方案在其他情况下也是十分有用的，因此我们可以使用@encode编译器指令来获取它：

```
NSLog(@"%s",@encode(NSString *));
NSLog(@"%s",@encode(NSInteger));

//结果：
2018-10-04 16:02:09.429772+0800 Demo[2690:231491] @
2018-10-04 16:02:09.429952+0800 Demo[2690:231491] q
```
当给定一个类型时，@encode返回这个类型的字符串编码。这些类型可以是诸如int、指针这样的基本类型，也可以是结构体、类等类型。事实上，任何可以作为sizeof()操作参数的类型都可以用于@encode()。  

在Objective-C Runtime Programming Guide中的[Type Encoding](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)一节中，列出了Objective-C中所有的类型编码。需要注意的是这些类型很多是与我们用于存档和分发的编码类型是相同的。但有一些不能在存档时使用。

>Objective-C不支持long double类型。@encode(long double)返回d，与double是一样的。

一个数组的类型编码位于方括号中；其中包含数组元素的个数及元素类型。如以下示例：  

```
float a[] = {1.0, 2.0, 3.0};
NSLog(@"array encoding type: %s", @encode(typeof(a)));  
NSNumber *b[] = {@1, @2, @3, @4};
NSLog(@"array encoding type: %s", @encode(typeof(b)));

//结果：
Demo[2858:246573]  array encoding type: [3f]
Demo[2858:246573]  array encoding type: [4@]
```
另外，还有些编码类型，@encode虽然不会直接返回它们，但它们可以作为协议中声明的方法的类型限定符。可以参考[Type Encoding](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)。

对于属性而言，还会有一些特殊的类型编码，以表明属性是只读、拷贝、retain等等，详情可以参考[Property Type String](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW6)。

### 成员变量、属性
Runtime中关于成员变量和属性的相关数据结构并不多，只有三个，并且都很简单。不过还有个非常实用但可能经常被忽视的特性，即关联对象，我们将在这小节中详细讨论。  

### 基础数据类型 

- Ivar  
Ivar是表示实例变量的类型，其实际上是一个指向objc_ivar结构体的指针，定义如下：  

```
struct objc_ivar {
    char * _Nullable ivar_name  OBJC2_UNAVAILABLE;
    char * _Nullable ivar_type  OBJC2_UNAVAILABLE;
    int ivar_offset  OBJC2_UNAVAILABLE; // 基地址偏移字节                                         
#ifdef __LP64__
    int space  OBJC2_UNAVAILABLE;
#endif
} 
```

-  objc_property_t  
objc_property_t是表示Objective-C声明的属性的类型，其实际是指向objc_property结构体的指针，其定义如下：

```
typedef struct objc_property *objc_property_t;

```  
objc_property_attribute_t定义了属性的特性(attribute)，它是一个结构体，定义如下:

```
typedef struct {
	constchar *name;// 特性名
	constchar *value;// 特性值
} objc_property_attribute_t;
```

- 关联对象(Associated Object)   
关联对象是Runtime中一个非常实用的特性，不过可能很容易被忽视。  

	关联对象类似于成员变量，不过是在运行时添加的。我们通常会把成员变量(Ivar)放在类声明的头文件中，或者放在类实现的@implementation后面。但这有一个缺点，我们不能在分类中添加成员变量。如果我们尝试在分类中添加新的成员变量，编译器会报错。
	
	我们可能希望通过使用(甚至是滥用)全局变量来解决这个问题。但这些都不是Ivar，因为他们不会连接到一个单独的实例。因此，这种方法很少使用。

	Objective-C针对这一问题，提供了一个解决方案：即关联对象(Associated Object)。

	我们可以把关联对象想象成一个Objective-C对象(如字典)，这个对象通过给定的key连接到类的一个实例上。不过由于使用的是C接口，所以key是一个void指针(const void *)。我们还需要指定一个内存管理策略，以告诉Runtime如何管理这个对象的内存。这个内存管理的策略可以由以下值指定：
>OBJC_ ASSOCIATION_ ASSIGN
>
>OBJC_ ASSOCIATION_ RETAIN_ NONATOMIC
>
>OBJC_ ASSOCIATION_ COPY_ NONATOMIC
>
>OBJC_ ASSOCIATION_ RETAIN
>
>OBJC_ ASSOCIATION_ COPY

当宿主对象被释放时，会根据指定的内存管理策略来处理关联对象。如果指定的策略是assign，则宿主释放时，关联对象不会被释放；而如果指定的是retain或者是copy，则宿主释放时，关联对象会被释放。我们甚至可以选择是否是自动retain/copy。当我们需要在多个线程中处理访问关联对象的多线程代码时，这就非常有用了。

我们将一个对象连接到其它对象所需要做的就是下面两行代码：
	
```	
static char myKey;
objc_setAssociatedObject(self, &myKey, anObject, OBJC_ASSOCIATION_RETAIN);
```
在这种情况下，self对象将获取一个新的关联的对象anObject，且内存管理策略是自动retain关联对象，当self对象释放时，会自动release关联对象。另外，如果我们使用同一个key来关联另外一个对象时，也会自动释放之前关联的对象，这种情况下，先前的关联对象会被妥善地处理掉，并且新的对象会使用它的内存。
	
```
id anObject = objc_getAssociatedObject(self, &myKey);	
``` 
我们可以使用objc_removeAssociatedObjects函数来移除一个关联对象，或者使用objc_setAssociatedObject函数将key指定的关联对象设置为nil。

### 成员变量、属性的操作方法
- 成员变量  
成员变量操作包含以下函数:

```
// 获取成员变量名
const char * ivar_getName (Ivar v);

// 获取成员变量类型编码
const char * ivar_getTypeEncoding (Ivar v);

// 获取成员变量的偏移量
ptrdiff_t ivar_getOffset (Ivar v);
```
ivar_getOffset函数，对于类型id或其它对象类型的实例变量，可以调用object_getIvar和object_setIvar来直接访问成员变量，而不使用偏移量。  

- 属性  
属性操作相关函数包括以下：

```
// 获取属性名
const char * property_getName (objc_property_t property);

// 获取属性特性描述字符串
const char * property_getAttributes (objc_property_t property);

// 获取属性中指定的特性
char * property_copyAttributeValue (objc_property_t property, const char *attributeName);

// 获取属性的特性列表
objc_property_attribute_t * property_copyAttributeList (objc_property_t property, unsigned int *outCount);
```
property_copyAttributeValue函数，返回的char *在使用完后需要调用free()释放。 property_copyAttributeList函数，返回值在使用完后需要调用free()释放。


- 关联对象  
关联对象操作函数包括以下：  

```
// 设置关联对象
void objc_setAssociatedObject (id object, const void *key, id value, objc_AssociationPolicy policy);

// 获取关联对象
id objc_getAssociatedObject (id object, const void *key);

// 移除关联对象
void objc_removeAssociatedObjects (id object);
```

[南峰子的技术博客](http://southpeak.github.io/blog/2014/10/30/objective-c-runtime-yun-xing-shi-zhi-er-:cheng-yuan-bian-liang-yu-shu-xing/)
