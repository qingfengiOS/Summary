##前言

###1.静态库和动态库有什么异同？

静态库：链接时完整地拷贝至可执行文件中，被多次使用就有多份冗余拷贝。利用静态函数库编译成的文件比较大，因为整个 函数库的所有数据都会被整合进目标代码中，他的优点就显而易见了，即编译后的执行程序不需要外部的函数库支持，因为所有使用的函数都已经被编译进去了。当然这也会成为他的缺点，因为如果静态函数库改变了，那么你的程序必须重新编译。

动态库：链接时不复制，程序运行时由系统动态加载到内存，供程序调用，系统只加载一次，多个程序共用，节省内存。由于函数库没有被整合进你的程序，而是程序运行时动态的申请并调用，所以程序的运行环境中必须提供相应的库。动态函数库的改变并不影响你的程序，所以动态函数库的升级比较方便。

静态库和动态库都是闭源库，只能拿来满足某个功能的使用，不会暴露内部具体的代码信息，而从github上下载的第三方库大多是开源库

静态库和动态库都是由*.o目标文件生成

使用静态库的好处：

- 模块化，分工合作  
- 避免少量改动经常导致大量的重复编译连接  
- 也可以重用，注意不是共享使用  

动态库使用有如下好处：

- 可以将最终可执行文件体积缩小
- 多个应用程序共享内存中得同一份库文件，节省资源  
- 可以不重新编译连接可执行程序的前提下，更新动态库文件达到更新应用程序的目的。
- 将整个应用程序分模块，团队合作，进行分工，影响比较小。

其实动态库应该叫共享库，那么从这个意义上来说，苹果禁止iOS开发中使用动态库就可以理解了： 因为在现在的iPhone，iPodTouch，iPad上面程序都是单进程的，也就是某一时刻只有一个进程在运行，那么你写个共享库

    --共享给谁？（你使用的时候只有你一个应用程序存在，其他的应该被挂起了，即便是可以同时多个进程运行，别人能使用你的共享库里的东西吗？你这个是给你自己的程序定制的。）
    --目前苹果的AppStore不支持模块更新，无法更新某个单独文件(除非自己写一个更新机制：有自己的服务端放置最新动态库文件)
至于苹果为啥禁止ios开发使用动态库我就猜到上面俩原因

2 这两种库都有哪些文件格式？

静态库：.a和.framework (windows:.lib , linux: .a)

动态库：.dylib和.framework（系统提供给我们的framework都是动态库！）(windows:.dll , linux: .so)

注意：两者都有framework的格式，但是当你创建一个framework文件时，系统默认是动态库的格式，如果想做成静态库，需要在buildSetting中将Mach-O Type选项设置为Static Library就行了！

3..a文件和.framework文件的区别？

- .a是一个纯二进制文件，不能直接拿来使用，需要配合头文件、资源文件一起使用。

- 将静态库打包的时候，只能打包代码资源，图片、本地json文件和xib等资源文件无法打包进去，使用.a静态库的时候需要三个组成部分：.a文件+需要暴露的头文件+资源文件；

- .framework中除了有二进制文件之外还有资源文件，可以拿来直接使用。

4.制作静态库需要注意的几点：

- 注意理解：无论是.a静态库还.framework静态库，我们需要的都是二进制文件+.h+其它资源文件的形式，不同的是，.a本身就是二进制文件，需要我们自己配上.h和其它文件才能使用，而.framework本身已经包含了.h和其它文件，可以直接使用。
- 图片资源的处理：两种静态库，一般都是把图片文件单独的放在一个.bundle文件中，一般.bundle的名字和.a或.framework的名字相同。.bundle文件很好弄，新建一个文件夹，把它改名为.bundle就可以了，右键，显示包内容可以向其中添加图片资源。
category是我们实际开发项目中经常用到的，把category打成静态库是没有问题的，但是在用这个静态库的工程中，调用category中的方法时会有找不到该方法的运行时错误（selector not recognized），解决办法是：在使用静态库的工程中配置other linkerflags的值为-ObjC。
- 如果一个静态库很复杂，需要暴露的.h比较多的话，就可以在静态库的内部创建一个.h文件（一般这个.h文件的名字和静态库的名字相同），然后把所有需要暴露出来的.h文件都集中放在这个.h文件中，而那些原本需要暴露的.h都不需要再暴露了，只需要把.h暴露出来就可以了。  

5.framework动态库的主要作用：

framework本来是苹果专属的内部提供的动态库文件格式，但是自从2014年WWDC之后，开发者也可以自定义创建framework实现动态更新（绕过apple store审核，从服务器发布更新版本）的功能，这与苹果限定的上架的app必须经过apple store的审核制度是冲突的，所以含有自定义的framework的app是无法在商店上架的，但是如果开发的是企业内部应用，就可以考虑尝试使用动态更新技术来将多个独立的app或者功能模块集成在一个app上面！（笔者开发的就是企业内部使用的app，我们将企业官网中的板块开发成4个独立的app，然后将其改造为framework文件集成在一款平台级的app当中进行使用）

目前 iOS 上的动态更新方案主要有以下 4 种：

HTML 5  
lua（wax）hotpatch  
react native  
framework  
前面三种都是通过在应用内搭建一个运行环境来实现动态更新（HTML 5 是原生支持），在用户体验、与系统交互上有一定的限制，对开发者的要求也更高（至少得熟悉 lua 或者 js）。

使用 framework 的方式来更新可以不依赖第三方库，使用原生的 OC/Swift 来开发，体验更好，开发成本也更低。

由于 Apple 不希望开发者绕过 App Store 来更新 app，因此只有对于不需要上架的应用，才能以 framework 的方式实现 app 的更新。

##主要思路

将 app 中的某个模块（比如一个 tab）的内容独立成一个 framework 的形式动态加载，在 app 的 main bundle 中，当 app 启动时从服务器上下载新版本的 framework 并加载即可达到动态更新的目的。

##实战

创建一个普通工程 DynamicUpdateDemo，其包含一个 framework 子工程 Module。也可以将 Module 创建为独立的工程，创建工程的过程不再赘述。

##依赖

在主工程的 Build Phases > Target Dependencies 中添加 Module，并且添加一个 New Copy Files Phase。

![](https://upload-images.jianshu.io/upload_images/2598795-b63ebaf0eefc348c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这样，打包时会将生成的 Module.framework 添加到 main bundle 的根目录下。

##使用动态库

通过以下方式将刚生成的framework添加到工程中：

Targets-->Build Phases-->Link Binary With Libraries
同时设置将framework作为资源文件拷贝到Bundle中：

Targets-->Build Phases-->Copy Bundle Resources
如图所示：

![](https://upload-images.jianshu.io/upload_images/2598795-3d8b89d72b4784a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


仅仅这样做是不够的，还需要为动态库添加链接依赖。

###自动链接动态库

添加完动态库后，如果希望动态库在软件启动时自动链接，可以通过以下方式设置动态库依赖路径：

Targets-->Build Setting-->Linking-->Runpath Search Paths
由于向工程中添加动态库时，将动态库设置了Copy Bundle Resources，因此就可以将Runpath Search Paths路径依赖设置为main bundle，即沙盒中的FrameworkDemo.app目录，向Runpath Search Paths中添加下述内容：

@executable_path/
如图所示：

![](https://upload-images.jianshu.io/upload_images/2598795-8500ac56b3e51c92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其中的@executable_path/表示可执行文件所在路径，即沙盒中的.app目录，注意不要漏掉最后的/。

如果你将动态库放到了沙盒中的其他目录，只需要添加对应路径的依赖就可以了。

###需要的时候链接动态库 
动态库的另一个重要特性就是即插即用性，我们可以选择在需要的时候再加载动态库。

- 更改设置
如果不希望在软件一启动就加载动态库，需要将

Targets-->Build Phases-->Link Binary With Libraries
中Dylib.framework对应的Status由默认的Required改成Optional；或者更干脆的，将Dylib.framework从Link Binary With Libraries列表中删除即可。

- 使用dlopen加载动态库
以Dylib.framework为例，动态库中真正的可执行代码为Dylib.framework/Dylib文件，因此使用dlopen时如果仅仅指定加载动态库的路径为Dylib.framework是没法成功加载的。

示例代码如下：  

```
- (IBAction)onDlopenLoadAtPathAction1:(id)sender
{
    NSString *documentsPath = [NSString stringWithFormat:@"%@/Documents/Dylib.framework/Dylib",NSHomeDirectory()];
    [self dlopenLoadDylibWithPath:documentsPath];
}

- (void)dlopenLoadDylibWithPath:(NSString *)path
{
    libHandle = NULL;
    libHandle = dlopen([path cStringUsingEncoding:NSUTF8StringEncoding], RTLD_NOW);
    if (libHandle == NULL) {
        char *error = dlerror();
        NSLog(@"dlopen error: %s", error);
    } else {
        NSLog(@"dlopen load framework success.");
    }
}

```
以dlopen方式使用动态库应该不能通过苹果审核。

- 使用NSBundle加载动态库
也可以使用NSBundle来加载动态库，实现代码如下：

```
- (IBAction)onBundleLoadAtPathAction1:(id)sender
{
    NSString *documentsPath = [NSString stringWithFormat:@"%@/Documents/Dylib.framework",NSHomeDirectory()];
    [self bundleLoadDylibWithPath:documentsPath];
}

- (void)bundleLoadDylibWithPath:(NSString *)path
{
    _libPath = path;
    NSError *err = nil;
    NSBundle *bundle = [NSBundle bundleWithPath:path];
    if ([bundle loadAndReturnError:&err]) {
        NSLog(@"bundle load framework success.");
    } else {
        NSLog(@"bundle load framework err:%@",err);
    }
}

```

###使用动态库中代码

通过上述任一一种方式加载的动态库后，就可以使用动态库中的代码文件了，以Dylib.framework中的Person类的使用为例：


```
- (IBAction)onTriggerButtonAction:(id)sender
{
    Class rootClass = NSClassFromString(@"Person");
    if (rootClass) {
        id object = [[rootClass alloc] init];
        [(Person *)object run];
    }
}

```

注意，如果直接通过下属方式初始化Person类是不成功的：

```
- (IBAction)onTriggerButtonAction:(id)sender
{
    Person *object = [[Person alloc] init];
    if (object) {
       [object run];
    }
}

```
##监测动态库的加载和移除

我们可以通过下述方式，为动态库的加载和移除添加监听回调：

```
+ (void)load
{
  _dyld_register_func_for_add_image(&image_added);
  _dyld_register_func_for_remove_image(&image_removed);
}

```
github上有一个完整的[示例代码](https://github.com/ddeville/ImageLogger)，

从这里看出，原来就算空白工程软件启动的时候也会加载多达一百二十多个动态库，如果这些都是静态库，那该有多可怕！！

##加载

主要的代码如下：

```
- (UIViewController *)loadFrameworkNamed:(NSString *)bundleName {
    NSArray* paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentDirectory = nil;
    if ([paths count] != 0) {
        documentDirectory = [paths objectAtIndex:0];
    }

    NSFileManager *manager = [NSFileManager defaultManager];
    NSString *bundlePath = [documentDirectory stringByAppendingPathComponent:[bundleName stringByAppendingString:@".framework"]];

    // Check if new bundle exists
    if (![manager fileExistsAtPath:bundlePath]) {
        NSLog(@"No framework update");
        bundlePath = [[NSBundle mainBundle]
                      pathForResource:bundleName ofType:@"framework"];

        // Check if default bundle exists
        if (![manager fileExistsAtPath:bundlePath]) {
            UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Oooops" message:@"Framework not found" delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil, nil];
            [alertView show];
            return nil;
        }
    }

    // Load bundle
    NSError *error = nil;
    NSBundle *frameworkBundle = [NSBundle bundleWithPath:bundlePath];
    if (frameworkBundle && [frameworkBundle loadAndReturnError:&error]) {
        NSLog(@"Load framework successfully");
    }else {
        NSLog(@"Failed to load framework with err: %@",error);
        return nil;
    }

    // Load class
    Class PublicAPIClass = NSClassFromString(@"PublicAPI");
    if (!PublicAPIClass) {
        NSLog(@"Unable to load class");
        return nil;
    }

    NSObject *publicAPIObject = [PublicAPIClass new];
    return [publicAPIObject performSelector:@selector(mainViewController)];
}
```
代码先尝试在 Document 目录下寻找更新后的 framework，如果没有找到，再在 main bundle 中寻找默认的 framework。 其中的关键是利用 OC 的动态特性 NSClassFromString 和 performSelector 加载 framework 的类并且执行其方法。

##framework 和 host 工程资源共用

###第方三库

Class XXX is implemented in both XXX and XXX. One of the two will be used. Which one is undefined.
这是当 framework 工程和 host 工程链接了相同的第三方库或者类造成的。

为了让打出的 framework 中不包含 host 工程中已包含的三方库（如 cocoapods 工程编译出的 .a 文件），可以这样：

删除 Build Phases > Link Binary With Libraries 中的内容（如有）。此时编译会提示三方库中包含的符号找不到。

在 framework 的 Build Settings > Other Linker Flags 添加 -undefined dynamic_lookup。必须保证 host 工程编译出的二进制文件中包含这些符号。

###类文件

尝试过在 framework 中引用 host 工程中已有的文件，通过 Build Settings > Header Search Paths 中添加相应的目录，Xcode 在编译的时候可以成功（因为添加了 -undefined dynamic_lookup），并且 Debug 版本是可以正常运行的，但是 Release 版本动态加载时会提示找不到符号：

```
Error Domain=NSCocoaErrorDomain Code=3588 "The bundle “YourFramework” couldn’t be loaded." (dlopen(/var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework, 265): Symbol not found: _OBJC_CLASS_$_BorderedView
      Referenced from: /var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework
      Expected in: flat namespace
     in /var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework) UserInfo=0x174276900 {NSLocalizedFailureReason=The bundle couldn’t be loaded., NSLocalizedRecoverySuggestion=Try reinstalling the bundle., NSFilePath=/var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework, NSDebugDescription=dlopen(/var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework, 265): Symbol not found: _OBJC_CLASS_$_BorderedView
      Referenced from: /var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework
      Expected in: flat namespace
     in /var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework/YourFramework, NSBundlePath=/var/mobile/Containers/Bundle/Application/5691FB75-408A-4D9A-9347-BC7B90D343C1/YourApp.app/YourFramework.framework, NSLocalizedDescription=The bundle “YourFramework” couldn’t be loaded.
    
```
因为 Debug 版本暴露了所有自定义类的符号以便于调试，因此你的 framework 可以找到相应的符号，而 Release 版本则不会。

目前能想到的方法只有将相同的文件拷贝一份到 framework 工程里，并且更改类名。

###访问 framework 中的图片

在 storyboard/xib 中可以直接访问图片，代码中访问的方法如下：

```
UIImage *image = [UIImage imageNamed:@"YourFramework.framework/imageName"]
```
注意：使用代码方式访问的图片不可以放在 xcassets 中，否则得到的将是 nil。并且文件名必须以 @2x/@3x 结尾，大小写敏感。因为 imageNamed: 默认在 main bundle 中查找图片。

###常见错误

###Architecture

```
dlopen(/path/to/framework, 9): no suitable image found.  Did find:
/path/to/framework: mach-o, but wrong architecture
```
这是说 framework 不支持当前机器的架构。 通过

```
lipo -info /path/to/MyFramework.framework/MyFramework
```
可以查看 framework 支持的 CPU 架构。

碰到这种错误，一般是因为编译 framework 的时候，scheme 选择的是模拟器，应该选择iOS Device。

此外，如果没有选择iOS Device，编译完成后，Products 目录下的 .framework 文件名会一直是红色，只有在 Derived Data 目录下才能找到编译生成的 .framework 文件。

###关于other linker flag

使用静态库或者动态库的时候极易发生链接错误，而且大多发生在加载framework中category的情况！根本原因在于Objective-C的链接器并不会为每个方法建立符号表，而是仅仅为类建立了符号表。这样的话，如果静态库中定义了已存在的一个类的分类，链接器就会以为这个类已经存在，不会把分类和核心类的代码合起来。这样的话，在最后的可执行文件中，就会缺少分类里的代码，这样函数调用就失败了。常见的设置方法就是在other linker flag中添加一个语句：-all_load，但是这样也并不是万能的，具体解析请参考链接：[链接](http://my.oschina.net/u/728866/blog/194741)

注意：当flag里面添加了注释却还是无法使用的时候，可能报flag与bitcode冲突的问题尤其是第三方库可能和bitcode冲突），这样的话就需要将bitcode设置为NO！

bitcode的具体作用不做详谈，可参考：[bitcode](http://www.jianshu.com/p/3e1b4e2d06c6)

###签名

系统在加载动态库时，会检查 framework 的签名，签名中必须包含 TeamIdentifier 并且 framework 和 host app 的 TeamIdentifier 必须一致。

如果不一致，否则会报下面的错误：

```
Error loading /path/to/framework: dlopen(/path/to/framework, 265): no suitable image found. Did find:
/path/to/framework: mmap() error 1
```
此外，如果用来打包的证书是 iOS 8 发布之前生成的，则打出的包验证的时候会没有 TeamIdentifier 这一项。这时在加载 framework 的时候会报下面的错误：

```
[deny-mmap] mapped file has no team identifier and is not a platform binary:
/private/var/mobile/Containers/Bundle/Application/5D8FB2F7-1083-4564-94B2-0CB7DC75C9D1/YourAppNameHere.app/Frameworks/YourFramework.framework/YourFramework
```
可以通过 codesign 命令来验证。

codesign -dv /path/to/YourApp.app
如果证书太旧，输出的结果如下：

```
Executable=/path/to/YourApp.app/YourApp
Identifier=com.company.yourapp
Format=bundle with Mach-O thin (armv7)
CodeDirectory v=20100 size=221748 flags=0x0(none) hashes=11079+5 location=embedded
Signature size=4321
Signed Time=2015年10月21日 上午10:18:37
Info.plist entries=42
TeamIdentifier=not set
Sealed Resources version=2 rules=12 files=2451
Internal requirements count=1 size=188

```
注意其中的 TeamIdentifier=not set。

采用 swift 加载 libswiftCore.dylib 这个动态库的时候也会遇到这个问题，对此Apple 官方的解释是：

```
To correct this problem, you will need to sign your app using code signing certificates with the Subject Organizational Unit (OU) set to your Team ID. All Enterprise and standard iOS developer certificates that are created after iOS 8 was released have the new Team ID field in the proper place to allow Swift language apps to run.

If you are an in-house Enterprise developer you will need to be careful that you do not revoke a distribution certificate that was used to sign an app any one of your Enterprise employees is still using as any apps that were signed with that enterprise distribution certificate will stop working immediately.
```
只能通过重新生成证书来解决这个问题。但是 revoke 旧的证书会使所有用户已经安装的，用该证书打包的 app 无法运行。

等等，我们就跪在这里了吗？！

现在企业证书的有效期是三年，当证书过期时，其打包的应用就不能运行，那企业应用怎么来更替证书呢？

__Apple 为每个账号提供了两个证书，这两个证书可以同时生效，这样在正在使用的证书过期之前，可以使用另外一个证书打包发布，让用户升级到新版本。__

也就是说，可以使用另外一个证书来打包应用，并且可以覆盖安装使用旧证书打包的应用。详情可以看 Apple 文档。

###深入理解iPhone静态库

在实际的编程过程中，通常会把一些公用函数制成函数库，供其它程序使用，一则提搞了代码的复用；二则提搞了核心技术的保密程度。所以在实际的项目开发中，经常会使用到函数库，函数库分为静态库和动态库两种。和多数人所熟悉的动态语言和静态语言一样，这里的所谓静态和动态是相对编译期和运行期的：静态库在程序编译时会被链接到目标代码中，程序运行时将不再需要改静态库；而动态库在程序编译时并不会被链接到目标代码中，只是在程序运行时才被载入，因为在程序运行期间还需要动态库的存在。

###深入理解framework（框架，相当于静态框架，不是动态库）

打包framework还是一个比较重要的功能，可以用来做一下事情：

封装功能模块，比如有比较成熟的功能模块封装成一个包，然后以后自己或其他同事用起来比较方便。
封装项目，有时候会遇到这个情况，就是一家公司找了两个开发公司做两个项目，然后要求他们的项目中的一个嵌套进另一个项目，此时也可以把呗嵌套的项目打包成framework放进去，这样比较方便。
###我们为什么需要框架（Framework）？

要想用一种开发者友好的方式共享库是很麻烦的。你不仅仅需要包含库本身，还要加入所有的头文件，资源等等。

苹果解决这个问题的方式是框架（framework）。基本上，这是含有固定结构并包含了引用该库时所必需的所有东西的文件夹。不幸的是，iOS禁止所有的动态库。同时，苹果也从Xcode中移除了创建静态iOS框架的功能。

Xcode仍然可以支持创建框架的功能，重启这个功能，我们需要对Xcode做一些小小的改动。

把代码封装在静态框架是被app store所允许的。尽管形式不同，本质上它仍然是一种静态库。

###框架（Framework）的类别

大部分框架都是动态链接库的形式。因为只有苹果才能在iOS设备上安装动态库，所以我们无法创建这种类型的框架。

静态链接库和动态库一样，只不过它是在编译时链接二进制代码，因此使用静态库不会有动态库那样的问题（即除了苹果谁也不能在iOS上使用动态库）。

“伪”框架是通过破解Xcode的目标Bundle（使用某些脚本）来实现的。它在表面上以及使用时跟静态框架并无区别。“伪”框架项目的功能几乎和真实的框架项目没有区别（不是全部）。

“嵌入”框架是静态框架的一个包装，以便Xcode能获取框架内的资源（图片、plist、nib等）。 本次发布包括了创建静态框架和“伪”框架的模板，以及二者的“嵌入”框架。

__用哪一种模板？__

本次发布有两个模板，每个模板都有“强”“弱”两个类别。你可以选择最适合一种（或者两种都安装上）。 最大的不同是Xcode不能创建“真”框架，除非你安装静态框架文件xcspec在Xcode中。这真是一个遗憾（这个文件是给项目使用的，而不是框架要用的）。

__简单说，你可以这样决定用哪一种模板：__

- 如果你不想修改Xcode，那么请使用“伪”框架版本
- 如果你只是想共享二进制（不是项目），两种都可以
- 如果你想把框架共享给不想修改Xcode的开发者，使用“伪”框架版本
- 如果你想把框架共享给修改过Xcode的开发者，使用“真”框架版本
- 如果你想把框架项目作为另一个项目的依赖（通过workspace或者子项目的方式），请使用“真”框架（或者“伪”框架，使用-framework——见后）
- 如果你想在你的框架项目中加入其他静态库／框架，并把它们也链接到最终结果以便不需要单独添加到用户项目中，使用“伪”框架  

__“伪”框架__

“伪”框架是破解的“reloacatable object file”（可重定位格式的目标文件， 保存着代码和数据，适合于和其他的目标文件连接到一起，用来创建一个可执行目标文件或者是一个可共享目标文件），它可以让Xcode编译出类似框架的东西——其实也是一个bundle。

“伪框架”模板把整个过程分为几个步骤，用某些脚本去产生一个真正的静态框架（基于静态库而不是reloacatable object file）。而且，框架项目还是把它定义为wrapper.cfbundle类型，一种Xcode中的“二等公民”。

因此它跟“真”静态框架一样可以正常工作，但当存在依赖关系时就有麻烦了。

__依赖问题__

如果不使用依赖，只是创建普通的项目是没有任何问题的。但是如果使用了项目依赖（比如在workspace中），Xcode就悲剧了。当你点击“Link Binary With Libraries”下方的’+’按钮时，“伪框架”无法显示在列表中。你可以从你的“伪”框架项目的Products下面将它手动拖入，但当你编辑你的主项目时，会出现警告：
```
warning: skipping file '/somewhere/MyFramework.framework' (unexpectedfile type 'wrapper.cfbundle' in Frameworks & Libraries build phase) 
```
并伴随“伪”框架中的链接错误。

幸运的是，有个办法来解决它。你可以在”Other Linker Flags”中用”-framwork”开关手动告诉linker去使用你的框架进行链接：

-framework MyFramework

警告仍然存在，但起码能正确链接了。

__添加其他的库/框架__

如果你加入其他静态（不是动态）库/框架到你的“伪”框架项目中，它们将“链接”进你最终的二进制框架文件中。在“真”框架项目中，它们是纯引用，而不是链接。

你可以在项目中仅仅包含头文件而不是静态库/框架本身的方式避免这种情况（以便编译通过）。

__“真”框架__

“真”框架各个方面都符合“真”的标准。它是真正的静态框架，正如使用苹果在从Xcode中去除的那个功能所创建的一样。

为了能创建真正的静态框架项目，你必需在Xcode中安装一个xcspec文件。

如果你发布一个“真”框架项目（而不是编译），希望去编译这个框架的人必需也安装xcspec文件（使用本次发布的安装脚本），以便Xcode能理解目标类型。

注意：如果你正在发布完全编译的框架，而不是框架项目，最终用户并不需要安装任何东西。 我已经提交一个报告给苹果，希望他们在Xcode中更新这个文件，但那需要一点时间。

__加其他静态库/框架__

如果你加入其他静态（不是动态）库/框架到你的“真”框架项目，它们只会被引用，而不会象“伪”框架一样被链接到最终的二进制文件中。

__从早期版本升级__

如果你是从Mk6或者更早的版本升级，同时使用“真”静态框架，并且使用Xcode4.2.1以前的版本，请运行uninstall_legacy.sh以卸载早期用于Xcode的所有修正。然后再运行install.sh，重启Xcode。如果你使用Xcode4.3以后，只需要运行install.sh并重启Xcode。

__安装__

分别运行Real Framework目录或Fake Framework目录下的install.sh脚本进行安装（或者两个你都运行）。

重启Xcode，你将在新项目向导的Framework&Library下看到StaticiOS Framework（或者Fake Static iOS Framework）。

卸载请运行unistall.sh脚本并重启Xcode。

__创建一个iOS框架项目__

1. 创建新项目。
2. 项目类型选择Framework&Library下的Static iOS Framework（或者Fake Static iOS Framework）。
3. 选择“包含单元测试”（可选的）。
4. 在target中加入类、资源等。
5. 凡是其他项目要使用的头文件，必需声明为public。进入target的Build Phases页，展开Copy Headers项，把需要public的头文件从Project或Private部分拖拽到Public部分。  
 
__编译你的 iOS 框架__

1. 选择指定target的scheme
2. 修改scheme的Run配置（可选）。Run配置默认使用Debug，但在准备部署的时候你可能想使用Release。
3. 编译框架（无论目标为iOS device和Simulator都会编译出相同的二进制，因此选谁都无所谓了）。
4. 从Products下选中你的framework，“show in Finder”。  
 
在build目录下有两个文件夹：(yourframework).framework and (your framework).embeddedframework.

如果你的框架只有代码，没有资源（比如图片、脚本、xib、coredata的momd文件等），你可以把(yourframework).framework 分发给你的用户就行了。如果还包含有资源，你必需分发(your framework).embeddedframework给你的用户。

为什么需要embedded framework？因为Xcode不会查找静态框架中的资源，如果你分发(your framework).framework, 则框架中的所有资源都不会显示，也不可用。

一个embedded framework只是一个framework之外的附加的包，包括了这个框架的所有资源的符号链接。这样做的目的是让Xcode能够找到这些资源。

__使用iOS 框架__

OS框架和常规的Mac OS动态框架差不多，只是它是静态链接的而已。

在你的项目中使用一个框架，只需把它拖仅你的项目中。在包含头文件时，记住使用尖括号而不是双引号括住框架名称。例如，对于框架MyFramework：

import <myframework myclass.h=""></myframework>

##使用问题

__Headers Not Found__

如果Xcode找不到框架的头文件，你可能是忘记将它们声明为public了。参考“创建一个iOS框架项目”第5步。

No Such Product Type

如果你没有安装iOS Universal Framework在Xcode，并企图编译一个universal框架项目（对于“真”框架，不是“假”框架），这会导致下列错误：

target specifies product type 'com.apple.product-type.framework.static',but there's no such product type for the 'iphonesimulator' platform

为了编译“真”iOS静态框架，Xcode需要做一些改动，因此为了编译“真”静态框架项目，请在所有的开发环境中安装它（对于使用框架的用户不需要，只有要编译框架才需要）。

__The selected run destination is not valid for this action__

有时，Xcode出错并加载了错误的active设置。首先，请尝试重启Xcode。如果错误继续存在，Xcode产生了一个坏的项目（因为Xcode4的一个bug，任何类型的项目都会出现这个问题）。如果是这样，你需要创建一个新项目重来一遍。

__链接警告__

第一次编译框架target时，Xcdoe会在链接阶段报告找不到文件夹： ld: warning: directory not found for option'-L/Users/myself/Library/Developer/Xcode/DerivedData/MyFramework-ccahfoccjqiognaqraesrxdyqcne/Build/Products/Debug-iphoneos' 此时，可以clean并重新编译target，警告会消除。

__Core Data momd not found__

对于框架项目和应用程序项目，Xcode会以不同的方式编译momd（托管对象模型文件）。Xcode会简单地在根目录创建.mom文件，而不会创建一个.momd目录（目录中包含VersionInfo.plist和.mom文件）。

这意味着，当从一个embedded framework的model中实例化NSManagedObjectModel时，你必需使用.mom扩展名作为model的URL，而不是采用.momd扩展名。

NSURL *modelURL = [[NSBundle mainBundle]URLForResource:@"MyModel" withExtension:@"mom"];

__Unknown class MyClass in Interface Builder file.__

由于静态框架采用静态链接，linker会剔除所有它认为无用的代码。不幸的是，linker不会检查xib文件，因此如果类是在xib中引用，而没有在O-C代码中引用，linker将从最终的可执行文件中删除类。这是linker的问题，不是框架的问题（当你编译一个静态库时也会发生这个问题）。苹果内置框架不会发生这个问题，因为他们是运行时动态加载的，存在于iOS设备固件中的动态库是不可能被删除的。 有两个解决的办法：

- 让框架的最终用户关闭linker的优化选项，通过在他们的项目的Other Linker Flags中添加-ObjC和-all_load。
在框架的另一个类中加一个该类的代码引用。例如，假设你有个MyTextField类，被linker剔除了。假设你还有一个MyViewController，它在xib中使用了MyTextField，MyViewController并没有被剔除。你应该这样做：

在MyTextField中：

```
-(void)forceLinkerLoad_ {
}  
```
在MyViewController中：

```
+(void) initialize { 
	[MyTextField forceLinkerLoad_]; 
}
```

他们仍然需要添加-ObjC到linker设置，但不需要强制all_load了。

- 这种方法需要你多做一点工作，但却让最终用户避免在使用你的框架时关闭linker优化（关闭linker优化会导致object文件膨胀）。  

__unexpected file type 'wrapper.cfbundle' in Frameworks &Libraries build phase__

这个问题发生在把“假”框架项目作为workspace的依赖，或者把它当作子项目时（“真”框架项目没有这个问题）。尽管这种框架项目产生了正确的静态框架，但Xcode只能从项目文件中看出这是一个bundle，因此它在检查依赖性时发出一个警告，并在linker阶段跳过它。

你可以手动添加一个命令让linker在链接阶段能正确链接。在依赖你的静态框架的项目的OtherLinker Flags中加入：

-framework MyFramework

警告仍然存在, 但不会导致链接失败。

__Libraries being linked or not being linked into the finalframework__

很不幸， “真”框架和“假”框架模板在处理引入的静态库/框架的工作方式不同的。

“真”框架模板采用正常的静态库生成步骤，不会链接其他静态库/框架到最终生产物中。

“假”框架模板采用“欺骗”Xcode的手段，让它认为是在编译一个可重定位格式的目标文件，在链接阶段就如同编译一个可执行文件，把所有的静态代码文件链接到最终生成物中（尽管不会检查是否确实目标代码）。为了实现象“真”框架一样的效果，你可以只包含库/框架的头文件到你的项目中，而不需要包含库/框架本身。

__Unrecognized selector in (some class with a category method)__

如果你的静态库或静态框架包含了一个模块（只在类别代码中声明，没有类实现），linker会搞不清楚，并把代码从二进制文件中剔除。因为在最终生成的文件中没有这个方法，所以当调用这个类别中定义的方法时，会报一个“unrecognizedselector”异常。

要解决这个，在包含这个类别的模块代码中加一个“假的”类。linker发现存在完整的O-C类，会将类别代码链接到模块。

我写了一个头文件 LoadableCategory.h，以减轻这个工作量：

```
import "SomeConcreteClass+MyAdditions.h"

import "LoadableCategory.h"

MAKE_CATEGORIES_LOADABLE(SomeConcreteClass_MyAdditions);

@implementation SomeConcreteClass(MyAdditions) ... 

@end
```
在使用这个框架时，仍然还需要在Build Setting的Other Linker Flags中加入-ObjC。

__执行任何代码前单元测试崩溃__

如果你在Xcode4.3中创建静态框架（或库）target时，勾选了“withunit tests”，当你试图运行单元测试时，它会崩溃：

Thread 1: EXC_BAD_ACCESS (code=2, address=0x0) 0 0x00000000 --- 15 dyldbootstrap:start(...)

这是lldb中的一个bug。你可以用GDB来运行单元测试。编辑scheme，选择Test，在Info标签中将调试器Debugger从LLDB改为GDB。  
[原文](https://legacy.gitbook.com/book/leon_lizi/-framework-/details)