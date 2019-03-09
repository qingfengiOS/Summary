#OC消息转发机制  
## 前言
在上一篇[Runtime源码 方法调用的过程](https://www.jianshu.com/p/13d475db9c99)中我们了解了消息的响应过程，即  

1. 先缓存查找，若未找到
2. 接下来查找本类的方法列表查找，若未找到
3. 则递归继承体系查找父类的方法列表直到NSObject  

在第二第三步过程中，如果找到了则响应消息，并填充缓存，缓存是保存在元类上的，如果还找不到，则进入接下来的消息转发流程。  

消息转发流程也分为三个步骤，动态解析 -> 前端转发 -> 方法签名转发，流程如下：  

![消息转发](https://upload-images.jianshu.io/upload_images/2598795-16d6f15b5b06b0f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们结合一个实例来具体看看[传送门：QFMessageForwardDemo](https://github.com/qingfengiOS/QFMessageForwardDemo.git)
## 动态解析
第一步，当没找到方法时，你可以通过```+ (BOOL)resolveInstanceMethod:(SEL)sel ```和```+ (BOOL)resolveClassMethod:(SEL)sel```来添加实例方法和类方法  

这里我们在```ViewController```中直接调用了```QFPerson```的```run```方法，但是```QFPerson```并没有实现这个方法,所以动态添加了这个方法。

```
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    
    if (sel == @selector(run:age:)) {//第一步自己添加方法
        class_addMethod(self, sel, (IMP)newRun, "v@:@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```  

这个添加一个自定义的方法

```
void newRun(id self,SEL sel, NSString *str, NSInteger age) {//自定义方法实现
    NSLog(@"---run ok---%@---%ld",str,(long)age);
}
```
这里用到了runtime的动态添加方法，不熟悉的可以看看这个系列文章的前几篇 
，执行结果：  

```
QFMessageForwardDemo[18572:704661] ---run ok---hello---18
```
## 前端转发

如果第一步没有动态添加方法，则会进入转发的第二步，前端转发，所谓前端转发即是，本类没有实现这个方法，但是另外的一个类实现了这个方法，那么我们可以直接转发这条消息到另外的类实现调用。这里我们新建```QFOtherPerson```` 并且实现```- (void)run:(NSString *)name age:(NSInteger)age```方法  

```
- (void)run:(NSString *)name age:(NSInteger)age{
    NSLog(@"%@ 执行了 run name = %@,age = %ld",NSStringFromClass([self class]), name,age);
}
```
在```QFPerson```的实现文件中，实现下面的方法完成转发 

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
    return [[QFOtherPerson alloc]init];//第二步，前端转发
}
```
>ps:此时要先注释注释掉第一步的动态添加方法  

执行结果：  

```
QFMessageForwardDemo[19629:739692] QFOtherPerson 执行了 run name = hello,age = 18
```
## 签名转发  
第三步，也是最后的机会处理这条消息了，首先生成方法签名：  

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {//第三步，签名转发
    NSString *methodName = NSStringFromSelector(aSelector);
    if ([methodName isEqualToString:@"run:age:"]) {//是我们需要转发的run方法
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    } else {
        return [super methodSignatureForSelector:aSelector];
    }
}
```
关于invocation你熟悉的可以看[ResponderChain+Strategy+MVVM实现一个优雅的TableView](https://juejin.im/post/5bd6c734e51d45410c10eb54)这里面有完整的invocation调用方法的例子，这里就不做粘贴了。  

最后调用```-forwardInvocation:```指定target，完成转发：   

```
- (void)forwardInvocation:(NSInvocation *)anInvocation {//签名转发
    SEL selector = [anInvocation selector];//目标方法

    QFOtherPerson *other = [[QFOtherPerson alloc]init];//转发对象

    if ([other respondsToSelector:selector]) {//目标对象能相应此方法
        [anInvocation invokeWithTarget:other];
    } else {
        return [super forwardInvocation:anInvocation];
    }
}
```  
运行结果：  

```
QFMessageForwardDemo[31317:930989] QFOtherPerson 执行了 run name = hello,age = 18
```
以上就是方法转发的流程，如果以上三步都没有实现则会崩溃，显示一个很常见的异常**- unrecognized selector sent to instance 0x60000001b440'**   

关于之前的流程请参考：[Runtime源码 方法调用的过程](https://juejin.im/post/5bda5a18e51d45688561c2f6)
