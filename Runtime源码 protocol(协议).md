#Runtime源码 protocol(协议)
### 一、概述
协议定义了一个纲领性的接口，所有类都可以选择实现。它主要是用来定义一套对象之间的通信规则。protocol也是我们设计时常用的一个东西，相对于直接继承的方式，protocol则偏向于组合模式。他使得两个毫不相关的类能够相互通信，从而实现特定的目标。因为OC是单继承的，由于不支持多继承，所以很多时候都是用Protocol和Category来代替实现"多继承"。

### 二、底层实现
在objc4-723中protocol的定义如下：  

```
struct protocol_t : objc_object {
	const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;

    const char *demangledName();
    ...
};

```
我们可以看到protocol继承自objc_object，里面的字段基本算是清晰，主要结构如下：  

- mangledName 和 _demangledName  
这是来源于C++的name mangling（命名重整）技术，在C++里面用来区别重载是的函数。

- protocols  
它是protocol_list_t类型的指针，保存了这个协议所遵守的协议；

- instanceMethods  
实例方法列表

- calssMethods  
类方法列表  

- optionalInstanceMethods  
可选择实现的实例方法，在声明时用@optional关键字修饰的实例方法

- optionalClassMethods  
可选择实现的类方法，在声明时用@optional关键字修饰的类方法

- instanceProperties  
实例属性  

- _classProperties  
类属性，比较少见  

	```
	@property (nonatomic, strong) NSString *name;//通过实例调用
	@property (class, nonatomic, strong) NSString *className;//通过类名调用
	```
### 三、	常用方法  
- protocol_copyPropertyList  
runtime提供了两个方法：

```
objc_property_t *
protocol_copyPropertyList(Protocol *proto, unsigned int *outCount)
{
    return protocol_copyPropertyList2(proto, outCount, 
                                      YES/*required*/, YES/*instance*/);
}


objc_property_t *
protocol_copyPropertyList2(Protocol *proto, unsigned int *outCount, 
                           BOOL isRequiredProperty, BOOL isInstanceProperty)
{
    if (!proto  ||  !isRequiredProperty) {
        // Optional properties are not currently supported.
        if (outCount) *outCount = 0;
        return nil;
    }

    rwlock_reader_t lock(runtimeLock);

    property_list_t *plist = isInstanceProperty
        ? newprotocol(proto)->instanceProperties
        : newprotocol(proto)->classProperties();
    return (objc_property_t *)copyPropertyList(plist, outCount);
}

```	  
**// Optional properties are not currently supported.**   
这里指明了可选属性现在还不支持，这就是没有可选属性的原因  

- conformsToProtocol() 

```
+ (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = self; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}

- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}
```
两个方法都是遍历类的继承集体，调用``` class_conformsToProtocol ```方法，其实现如下：  

```
BOOL class_conformsToProtocol(Class cls, Protocol *proto_gen)
{
    protocol_t *proto = newprotocol(proto_gen);
    
    if (!cls) return NO;
    if (!proto_gen) return NO;

    rwlock_reader_t lock(runtimeLock);

    assert(cls->isRealized());

    for (const auto& proto_ref : cls->data()->protocols) {
        protocol_t *p = remapProtocol(proto_ref);
        if (p == proto || protocol_conformsToProtocol_nolock(p, proto)) {
            return YES;
        }
    }

    return NO;
}
```
把class的protocols取出来，并与传入的protocol做比较，如果地址相同直接返回，或者协议"继承"的层级中满足条件：  

```
/***********************************************************************
* protocol_conformsToProtocol_nolock
* Returns YES if self conforms to other.
* Locking: runtimeLock must be held by the caller.
**********************************************************************/
static bool 
protocol_conformsToProtocol_nolock(protocol_t *self, protocol_t *other)
{
    runtimeLock.assertLocked();

    if (!self  ||  !other) {
        return NO;
    }

    // protocols need not be fixed up

    if (0 == strcmp(self->mangledName, other->mangledName)) {
        return YES;
    }

    if (self->protocols) {
        uintptr_t i;
        for (i = 0; i < self->protocols->count; i++) {
            protocol_t *proto = remapProtocol(self->protocols->list[i]);
            if (0 == strcmp(other->mangledName, proto->mangledName)) {
                return YES;
            }
            if (protocol_conformsToProtocol_nolock(proto, other)) {
                return YES;
            }
        }
    }

    return NO;
}
```
递归处理，对比协议的mangledName，有相同的就返回YES。  

参考：  
[协议protocol](https://www.jianshu.com/p/fe8048524e67)  
[Protocol](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/Protocol.html)