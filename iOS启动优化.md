#启动优化
##pre-main阶段
exec() 是一个系统调用。系统内核把应用映射到新的地址空间，且每次起始位置都是随机的（因为使用 ASLR）。并将起始位置到 0x000000 这段范围的进程权限都标记为不可读写不可执行。如果是 32 位进程，这个范围至少是 4KB；对于 64 位进程则至少是 4GB。NULL 指针引用和指针截断误差都是会被它捕获。  

###1.Load dylibs
这一阶段dylib会分析应用依赖的dylib，找到mach-o文件，打开和读取这些文件并验证有效性，接着会找到代码签名注册到内核，最后都对dylib的每一个segment调用mmap（）。一般情况下，iOS应用会加载100-400个dylibs，其中大部分是系统库，这部分dylib的加载系统已经做了优化。

所以，依赖的dylib越少越好。在这一步，我们可以做的优化有：  

1.尽量不使用内嵌（embedded）的dylib，加载内嵌dylib性能开销较大
2.合并已有的dylib和使用静态库（static archives），减少dylib的使用个数  
3.懒加载dylib，但是要注意dlopen()可能造成一些问题，且实际上懒加载做的工作会更多
###2.Rebase/Bind
在dylib的加载过程中，为了安全考虑一如了ASLR（Address Space Layout Randomization）技术和代码签名。由于ASLR的原因，镜像（Image，包括可执行文件、dylib和bundle）会在随机的地址上加载dyld需要修正这个偏差，来指向正确的地址。  
Rebase在前，Bind在后，Rebase做的是将镜像读入内存，修正镜像内部的指针，性能消耗主要在IO。Bind做的是查询符号表，设置指向镜像外部的指针，性能消耗主要在CPU计算。  

所以，指针数量越少越好。在这一步，我们可以做的优化有：

1.减少ObjC类（class）、方法（selector）、分类（category）的数量  
2.减少C++虚函数的的数量（创建虚函数表有开销）  
3.使用Swift structs（内部做了优化，符号数量更少）
###3.ObjC setup
大部分ObjC初始化工作已经在Rebase/Bind阶段做完了，这一步dyld会注册所有声明过的ObjC类，将分类插入到类的方法列表里，再检查每个selector的唯一性。  

在这一步倒没什么优化可做的，Rebase/Bind阶段优化好了，这一步的耗时也会减少。
###4.initializers
到了这一阶段，dyld开始运行程序的初始化函数，调用每个ObjC类和分类的load方法，调用C/C++中的构造函数和创建非基本类型的C++静态全局变量。initializers阶段执行完后，dyld开始调用manin（）函数  

在这一步，我们可以做的优化有：

1.少在类的+load方法里做事情，尽量把这些事情推迟到+initiailize  
2.减少构造器函数个数，在构造器函数里少做些事情  
3.减少C++静态全局变量的个数
##main阶段
这一阶段的优化主要是减少didFinishLaunchingWithOptions方法里的工作，在didFinishLaunchingWithOptions方法里，我们会创建应用的window，指定其rootViewController，调用window的makeKeyAndVisible方法让其可见。由于业务需要，我们会初始化各个二方/三方库，设置系统UI风格，检查是否需要显示引导页、是否需要登录、是否有新版本等，由于历史原因，这里的代码容易变得比较庞大，启动耗时难以控制。

所以，满足业务需要的前提下，didFinishLaunchingWithOptions在主线程里做的事情越少越好。在这一步，我们可以做的优化有：

1.梳理各个二方/三方库，找到可以延迟加载的库，做延迟加载处理，比如放到首页控制器的viewDidAppear方法里。  
2.梳理业务逻辑，把可以延迟执行的逻辑，做延迟执行处理。比如检查新版本、注册推送通知等逻辑。  
3.避免复杂/多余的计算。  
4.避免在首页控制器的viewDidLoad和viewWillAppear做太多事情，这2个方法执行完，首页控制器才能显示，部分可以延迟创建的视图应做延迟创建/懒加载处理。  
5.采用性能更好的API。  
6.首页控制器用纯代码方式来构建。
