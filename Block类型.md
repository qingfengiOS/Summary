# Block的类型
### 前言 
Block在iOS日常开发中极其常见，大家应该几乎都使用过，比较熟悉它的用法，而且知道Block可能引起循环引用，今天来聊聊Block，以及Block造成内存泄露的根本原因。

### Block是什么
首先，Block和普通实例一样是是一个对象，他有自己的isa指针。  
它就是一个里面存储了指向定义代码块的函数指针和block外部上下文变量信息的结构体。通过断点我们看到block的isa指针，如下图：

![isa](https://user-gold-cdn.xitu.io/2019/2/23/16918a290bdf247f?w=736&h=548&f=png&s=135828)

我们发现block的类型其实是不同的，这是为什么接下来我们看看Block到底有哪些类型。 

### Block的类型
我们通过实际例子看看的各种类型的block
  
- NSMallocBlock  
	
```
- (void)NSMallocBlock {
    int tempInt = 1;
    void (^block)(void) = ^ {
        NSLog(@"----------%d----------\n\n",tempInt);
    };
    block();
    [self printBlockSuperClass:block];
}
```  

结果：__NSMallocBlock__ -> __NSMallocBlock -> NSBlock -> NSObject

- NSStaticBlock  

```
- (void)NSStaticBlock {
    int tempInt = 1;
    __weak void (^block)(void) = ^ {
        NSLog(@"----------%d----------\n\n",tempInt);
    };
    block();
    
    [self printBlockSuperClass:block];
}
```
结果：__NSStackBlock__ -> __NSStackBlock -> NSBlock -> NSObject


- NSGlobalBlock  

```
- (void)NSGlobalBlock {
    void (^block)(int a) = ^ (int a){
        NSLog(@"----------%d----------\n\n",a);
    };
    block(1);
    
    [self printBlockSuperClass:block];
}
```

结果：__NSGlobalBlock__ -> __NSGlobalBlock -> NSBlock -> NSObject

我们发现：

1. 当没有外部变量时，block为__NSMallocBlock，它由开发者创建，存储在堆内存上。
2. 当有```__weak```修饰时block为__NSStackBlock，存储在栈区。
3. 当block有参数时（捕获了外部变量时）block为__NSGlobalBlock，存储在全局区。  

### 属性关键字和外部变量类型对Block内存的影响  
为了验证我们定义了三中关键字的block,分别有storng、weak、copy修饰：  

```
@property (nonatomic, strong) TestBlock strongBlock;
@property (nonatomic, weak) TestBlock weakBlock;
@property (nonatomic, copy) TestBlock copyBlock;
```

验证方法如下：  

```
int globalInt = 1000;//全局变量
static staticInt = 10000;//全局静态变量

- (void)blockInMemory {
    static tempStaticInt = 100000;//局部静态变量
    int normalInt = 20000;
    _strongBlock = ^(int tempInt) {
        NSLog(@"tempInt = %d", normalInt);
    };
    _weakBlock = ^(int tempInt) {
        NSLog(@"tempInt = %d", normalInt);
    };
    _copyBlock = ^(int tempInt) {
        NSLog(@"tempInt = %d", normalInt);
    };
    NSLog(@"\nstrongBlock:%@\n_weakBlock:%@\n_copyBlock:%@",object_getClass(_strongBlock),object_getClass(_weakBlock),object_getClass(_copyBlock));
}
```
分别打印不同变量类型（全局变量、全局静态变量、局部静态变量、局部变量）和属性关键字下block的类型，我们可以得出如下结论：  

1. 没有外部变量时,三种Block都是 ```__NSGlobalBlock__```
2. 有外部变量时,   
    2.1 外部变量时全局变量、全局静态变量、局部静态变量时：```__NSGlobalBlock__ ```（全局区）  
    2.2 外部变量时普通外部变量：copy和strong修饰的Block是``` __NSMallocBlock__```（堆区）；weak修饰的block是```__NSStackBlock__```（栈区）
    
 >有普通外部变量的block是在栈区创建的，当有copy和strong修饰符修饰的时，会把block从栈移到堆区。  
 
 >ARC下使用copy和strong关键字修饰block是一样的。
 
### 结语
本篇为Block系列的第一篇，由此，我们了解了三种不同类型Block，接下来会以源码的方式深入了解block的底层实现，我们下篇再见。  
[Demo工程](https://github.com/qingfengiOS/BlockInMemory)