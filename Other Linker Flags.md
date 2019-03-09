#Other Linker Flags
##前言
一个程序从代码到执行文件需要经历如下几个阶段：  
__源代码->预处理->编译->汇编->机器码->链接->可执行文件（镜像文件）__  

Other Linker Flags到底是用来干嘛的呢？就是ld命令除了默认参数外的其他参数。ld命令实现的是链接的工作，详细说明可以在终端man ld查看。  

源文件经过一系列处理以后，会生成对应的.obj文件，然后一个项目必然会有许多.obj文件，并且这些文件之间会有各种各样的联系，例如函数调用。链接器做的事就是把这些目标文件和所用的一些库链接在一起形成一个完整的可执行文件。

##selector not recognized
苹果官方Q&A上有这么一段话：  

The "selector not recognized" runtime exception occurs due to an issue between the implementation of standard UNIX static libraries, the linker and the dynamic nature of Objective-C. Objective-C does not define linker symbols for each function (or method, in Objective-C) - instead, linker symbols are only generated for each class. If you extend a pre-existing class with categories, the linker does not know to associate the object code of the core class implementation and the category implementation. This prevents objects created in the resulting application from responding to a selector that is defined in the category.


翻译过来，大概意思就是Objective-C的链接器并不会为每个方法建立符号表，而是仅仅为类建立了符号表。这样的话，如果静态库中定义了已存在的一个类的分类，链接器就会以为这个类已经存在，不会把分类和核心类的代码合起来。这样的话，在最后的可执行文件中，就会缺少分类里的代码，这样函数调用就失败了。
 
##解决方法
解决方法在背景那块我就提到了，就是在Other Linker Flags里加上所需的参数，用到的参数一般有以下3个：

1. -ObjC 
2. -all_load
3. -force_load

-ObjC, 一般这个参数足够解决前面提到的问题，苹果官方说明如下：  
This flag causes the linker to load every object file in the library that defines an Objective-C class or category. While this option will typically result in a larger executable (due to additional object code loaded into the application), it will allow the successful creation of effective Objective-C static libraries that contain categories on existing classes.

简单说来，加了这个参数后，链接器就会把静态库中所有的Objective-C类和分类都加载到最后的可执行文件中，虽然这样可能会因为加载了很多不必要的文件而导致可执行文件变大，但是这个参数很好地解决了我们所遇到的问题。但是事实真的是这样的吗？

如果-ObjC参数真的这么有效，那么事情就会简单多了。

Important: For 64-bit and iPhone OS applications, there is a linker bug that prevents -ObjC from loading objects files from static libraries that contain only categories and no classes. The workaround is to use the -allload or -forceload flags.

当静态库中只有分类而没有类的时候，-ObjC参数就会失效了。这时候，就需要使用-all_load或者-force_load了。

-allload会让链接器把所有找到的目标文件都加载到可执行文件中，但是__千万不要随便使用这个参数！__假如你使用了不止一个静态库文件，然后又使用了这个参数，那么你很有可能会遇到ld: duplicate symbol错误，因为不同的库文件里面可能会有相同的目标文件，所以建议在遇到-ObjC失效的情况下使用-force_load参数。

-forceload所做的事情跟-all_load其实是一样的，但是-force_load需要指定要进行全部加载的库文件的路径，这样的话，你就只是完全加载了一个库文件，不影响其余库文件的按需加载。  
[原文](https://my.oschina.net/u/728866/blog/194741)
