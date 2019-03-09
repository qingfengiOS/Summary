#iOS编译过程

## 前言
iOS 开发中使用的是编译语言，所谓编译语言是在执行的时候，必须先通过编译器生成机器码，机器码可以直接在CPU上执行，所以执行效率较高。他是使用 Clang / LLVM 来编译的。LLVM是一个模块化和可重用的编译器和工具链技术的集合，Clang 是 LLVM 的子项目，是 C，C++ 和 Objective-C 编译器，目的是提供惊人的快速编译。下面我们来看看编译过程,总的来说编译过程分为几个阶段:  
**预处理 -> 词法分析 -> 语法分析 -> 静态分析 -> 生成中间代码和优化 -> 汇编 -> 链接**

## 具体过程

### 一、预处理  
我们以一个实际例子来看看，预处理的过程，源码：  

```
#import "AppDelegate.h"

#define NUMBER 1
int main(int argc, char * argv[]) {
    @autoreleasepool {
    
        NSLog(@"%d",NUMBER);
        
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```  
使用终端到main.m所在文件夹，使用命令：```clang -E main.m```,结果如下：  

```
@interface AppDelegate : UIResponder <UIApplicationDelegate>

@property (strong, nonatomic) UIWindow *window;

@end
# 11 "main.m" 2

int main(int argc, char * argv[]) {
    @autoreleasepool {

        NSLog(@"%d",1);

        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```  
也可以使用Xcode的```Product->Perform Action -> Preprocess```得到相同的结果  
这一步编译器所做的处理是：  

- 宏替换  
	在源码中使用的宏定义会被替换为对应#define的内容）
	  
	>建议大家不要在需要预处理的代码中加入内联代码逻辑。
- 头文件引入(#include，#import)  
	使用对应文件.h的内容替换这一行的内容,所以尽量减少头文件中的#import，使用@class替代，把#import放到.m文件中。	
- 处理条件编译指令 （#if,#else,#endif)  
	
### 二、词法解析
使用```clang -Xclang -dump-tokens main.m```词法分析结果如下：  

```
int 'int'	 [StartOfLine]	Loc=<main.m:14:1>
identifier 'main'	 [LeadingSpace]	Loc=<main.m:14:5>
l_paren '('		Loc=<main.m:14:9>
int 'int'		Loc=<main.m:14:10>
identifier 'argc'	 [LeadingSpace]	Loc=<main.m:14:14>
comma ','		Loc=<main.m:14:18>
char 'char'	 [LeadingSpace]	Loc=<main.m:14:20>
star '*'	 [LeadingSpace]	Loc=<main.m:14:25>
identifier 'argv'	 [LeadingSpace]	Loc=<main.m:14:27>
l_square '['		Loc=<main.m:14:31>
r_square ']'		Loc=<main.m:14:32>
r_paren ')'		Loc=<main.m:14:33>
l_brace '{'	 [LeadingSpace]	Loc=<main.m:14:35>
at '@'	 [StartOfLine] [LeadingSpace]	Loc=<main.m:15:5>
identifier 'autoreleasepool'		Loc=<main.m:15:6>
l_brace '{'	 [LeadingSpace]	Loc=<main.m:15:22>
identifier 'NSLog'	 [StartOfLine] [LeadingSpace]	Loc=<main.m:17:9>
l_paren '('		Loc=<main.m:17:14>
at '@'		Loc=<main.m:17:15>
string_literal '"%d"'		Loc=<main.m:17:16>
comma ','		Loc=<main.m:17:20>
numeric_constant '1'		Loc=<main.m:17:21 <Spelling=main.m:12:16>>
r_paren ')'		Loc=<main.m:17:27>
semi ';'		Loc=<main.m:17:28>
return 'return'	 [StartOfLine] [LeadingSpace]	Loc=<main.m:19:9>
identifier 'UIApplicationMain'	 [LeadingSpace]	Loc=<main.m:19:16>
l_paren '('		Loc=<main.m:19:33>
identifier 'argc'		Loc=<main.m:19:34>
comma ','		Loc=<main.m:19:38>
identifier 'argv'	 [LeadingSpace]	Loc=<main.m:19:40>
comma ','		Loc=<main.m:19:44>
identifier 'nil'	 [LeadingSpace]	Loc=<main.m:19:46>
comma ','		Loc=<main.m:19:49>
identifier 'NSStringFromClass'	 [LeadingSpace]	Loc=<main.m:19:51>
l_paren '('		Loc=<main.m:19:68>
l_square '['		Loc=<main.m:19:69>
identifier 'AppDelegate'		Loc=<main.m:19:70>
identifier 'class'	 [LeadingSpace]	Loc=<main.m:19:82>
r_square ']'		Loc=<main.m:19:87>
r_paren ')'		Loc=<main.m:19:88>
r_paren ')'		Loc=<main.m:19:89>
semi ';'		Loc=<main.m:19:90>
r_brace '}'	 [StartOfLine] [LeadingSpace]	Loc=<main.m:20:5>
r_brace '}'	 [StartOfLine]	Loc=<main.m:21:1>
eof ''		Loc=<main.m:21:2>
```
这一步把源文件中的代码转化为特殊的标记流，源码被分割成一个一个的字符和单词，在行尾Loc中都标记出了源码所在的对应源文件和具体行数，方便在报错时定位问题。
### 三、语法分析  
执行 ```clang 命令 clang -Xclang -ast-dump -fsyntax-only maim.m```得到如下结果：  

```
|-FunctionDecl 0x7f9fa085a9b8 <main.m:14:1, line:21:1> line:14:5 main 'int (int, char **)'
| |-ParmVarDecl 0x7f9fa085a788 <col:10, col:14> col:14 used argc 'int'
| |-ParmVarDecl 0x7f9fa085a8a0 <col:20, col:32> col:27 used argv 'char **':'char **'
| `-CompoundStmt 0x7f9fa1002240 <col:35, line:21:1>
|   `-ObjCAutoreleasePoolStmt 0x7f9fa1002230 <line:15:5, line:20:5>
|     `-CompoundStmt 0x7f9fa1002210 <line:15:22, line:20:5>
|       `-CallExpr 0x7f9fa085aec0 <line:17:9, col:27> 'void'
|         |-ImplicitCastExpr 0x7f9fa085aea8 <col:9> 'void (*)(id, ...)' <FunctionToPointerDecay>
|         | `-DeclRefExpr 0x7f9fa085ac90 <col:9> 'void (id, ...)' Function 0x7f9fa085ab38 'NSLog' 'void (id, ...)'
|         |-ImplicitCastExpr 0x7f9fa085aef8 <col:15, col:16> 'id':'id' <BitCast>
|         | `-ObjCStringLiteral 0x7f9fa085ae08 <col:15, col:16> 'NSString *'
|         |   `-StringLiteral 0x7f9fa085acf8 <col:16> 'char [3]' lvalue "%d"
|         `-IntegerLiteral 0x7f9fa085ae28 <line:12:16> 'int' 1
|-FunctionDecl 0x7f9fa085ab38 <line:17:9> col:9 implicit used NSLog 'void (id, ...)' extern
| |-ParmVarDecl 0x7f9fa085abd0 <<invalid sloc>> <invalid sloc> 'id':'id'
| `-FormatAttr 0x7f9fa085ac38 <col:9> Implicit NSString 1 2
|-FunctionDecl 0x7f9fa085af60 <<invalid sloc>> line:19:16 implicit used UIApplicationMain 'int ()'
`-FunctionDecl 0x7f9fa085b098 <<invalid sloc>> col:51 implicit used NSStringFromClass 'int ()'
```  
这一步是把词法分析生成的标记流，解析成一个抽象语法树（abstract [syntax tree -- AST](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)）,同样地，在这里面每一节点也都标记了其在源码中的位置。  
### 四、静态分析
把源码转化为抽象语法树之后，编译器就可以对这个树进行分析处理。静态分析会对代码进行错误检查，如出现方法被调用但是未定义、定义但是未使用的变量等，以此提高代码质量。当然，还可以通过使用 Xcode 自带的静态分析工具（Product -> Analyze)

- 类型检查  
	在此阶段clang会做检查，最常见的是检查程序是否发送正确的消息给正确的对象，是否在正确的值上调用了正常函数。如果你给一个单纯的 NSObject* 对象发送了一个 hello 消息，那么 clang 就会报错，同样，给属性设置一个与其自身类型不相符的对象，编译器会给出一个可能使用不正确的警告。  
	
	>一般会把类型分为两类：动态的和静态的。动态的在运行时做检查，静态的在编译时做检查。以往，编写代码时可以向任意对象发送任何消息，在运行时，才会检查对象是否能够响应这些消息。由于只是在运行时做此类检查，所以叫做动态类型。

	>至于静态类型，是在编译时做检查。当在代码中使用 ARC 时，编译器在编译期间，会做许多的类型检查：因为编译器需要知道哪个对象该如何使用。 
- 其他分析   
	```ObjCUnusedIVarsChecker.cpp```是用来检查是否有定义了，但是从未使用过的变量。    
	```ObjCSelfInitChecker.cpp```是检查在 你的初始化方法中中调用 self 之前，是否已经调用 [self initWith...] 或 [super init] 了。  
	
	![checkers](https://user-gold-cdn.xitu.io/2018/12/17/167bb9570949934d?w=1030&h=364&f=png&s=147454)
	更多请看:[clang静态分析](https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer)

### 五、中间代码生成和优化
使用```clang -O3 -S -emit-llvm main.m -o main.ll```生成main.ll文件，打开并查看转化结果：  

```
; ModuleID = 'main.m'
source_filename = "main.m"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.13.0"

%struct.__NSConstantString_tag = type { i32*, i32, i8*, i64 }

@__CFConstantStringClassReference = external global [0 x i32]
@.str = private unnamed_addr constant [3 x i8] c"%d\00", section "__TEXT,__cstring,cstring_literals", align 1
@_unnamed_cfstring_ = private global %struct.__NSConstantString_tag { i32* getelementptr inbounds ([0 x i32], [0 x i32]* @__CFConstantStringClassReference, i32 0, i32 0), i32 1992, i8* getelementptr inbounds ([3 x i8], [3 x i8]* @.str, i32 0, i32 0), i64 2 }, section "__DATA,__cfstring", align 8

; Function Attrs: ssp uwtable
define i32 @main(i32, i8** nocapture readnone) local_unnamed_addr #0 {
  %3 = tail call i8* @objc_autoreleasePoolPush() #2
  notail call void (i8*, ...) @NSLog(i8* bitcast (%struct.__NSConstantString_tag* @_unnamed_cfstring_ to i8*), i32 1)
  tail call void @objc_autoreleasePoolPop(i8* %3)
  ret i32 0
}

declare i8* @objc_autoreleasePoolPush() local_unnamed_addr

declare void @NSLog(i8*, ...) local_unnamed_addr #1

declare void @objc_autoreleasePoolPop(i8*) local_unnamed_addr

attributes #0 = { ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #2 = { nounwind }

!llvm.module.flags = !{!0, !1, !2, !3, !4, !5, !6}
!llvm.ident = !{!7}

!0 = !{i32 1, !"Objective-C Version", i32 2}
!1 = !{i32 1, !"Objective-C Image Info Version", i32 0}
!2 = !{i32 1, !"Objective-C Image Info Section", !"__DATA,__objc_imageinfo,regular,no_dead_strip"}
!3 = !{i32 4, !"Objective-C Garbage Collection", i32 0}
!4 = !{i32 1, !"Objective-C Class Properties", i32 64}
!5 = !{i32 1, !"wchar_size", i32 4}
!6 = !{i32 7, !"PIC Level", i32 2}
!7 = !{!"Apple LLVM version 9.1.0 (clang-902.0.39.2)"}
```  

接下来 LLVM 会对代码进行编译优化，例如针对全局变量优化、循环优化、尾递归优化等，最后输出汇编代码。  

使用```xcrun clang -S -o - main.m | open -f```生成汇编代码：  

```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 13
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## BB#0:
	pushq	%rbp
Lcfi0:
	.cfi_def_cfa_offset 16
Lcfi1:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Lcfi2:
	.cfi_def_cfa_register %rbp
	subq	$32, %rsp
	movl	$0, -4(%rbp)
	movl	%edi, -8(%rbp)
	movq	%rsi, -16(%rbp)
	callq	_objc_autoreleasePoolPush
	leaq	L__unnamed_cfstring_(%rip), %rsi
	movl	$1, %edi
	movl	%edi, -20(%rbp)         ## 4-byte Spill
	movq	%rsi, %rdi
	movl	-20(%rbp), %esi         ## 4-byte Reload
	movq	%rax, -32(%rbp)         ## 8-byte Spill
	movb	$0, %al
	callq	_NSLog
	movq	-32(%rbp), %rdi         ## 8-byte Reload
	callq	_objc_autoreleasePoolPop
	xorl	%eax, %eax
	addq	$32, %rsp
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
	.asciz	"%d"

	.section	__DATA,__cfstring
	.p2align	3               ## @_unnamed_cfstring_
L__unnamed_cfstring_:
	.quad	___CFConstantStringClassReference
	.long	1992                    ## 0x7c8
	.space	4
	.quad	L_.str
	.quad	2                       ## 0x2

	.section	__DATA,__objc_imageinfo,regular,no_dead_strip
L_OBJC_IMAGE_INFO:
	.long	0
	.long	64


.subsections_via_symbols

```
前面的三行: 

```
    .section    __TEXT,__text,regular,pure_instructions
    .macosx_version_min 10, 13
    .globl  _main                   ## -- Begin function main
    .p2align    4, 0x90
```
他们是汇编指令而不是汇编代码。  

- ```.section```指令指定了接下来会执行哪一个段
- ```.globl```指令说明```_main```是一个外部符号。这就是我们的main()函数。这个函数对外部是可见的，因为系统要调用它来运行可执行文件。  
- ```.p2align```指令指出了后面代码的对齐方式。在我们的代码中，后面的代码会按照 16(2^4) 字节对齐，如果需要的话，用 0x90 补齐。

想要了解更多可以看一下这篇文章：[《LLVM 全时优化》](https://blog.csdn.net/dashuniuniu/article/details/50385528)。

### 六、汇编
在这一阶段，汇编器将上一步生成的可读的汇编代码转化为机器代码。最终产物就是 以 .o 结尾的目标文件。使用Xcode构建的程序会在DerivedData目录中找到这个文件。如图：  

![.o](https://user-gold-cdn.xitu.io/2018/12/18/167c00b045143702?w=3934&h=572&f=png&s=605402)  

### 七、链接
这一阶段是将上个阶段生成的目标文件和引用的静态库链接起来，最终生成可执行文件,链接器解决了目标文件和库之间的链接。
  
使用```clang main.m```生成可执行文件a.out(不指定名字默认为a.out),使用```file a.out```可以看到其类型信息：  

```
a.out: Mach-O 64-bit executable x86_64
```
可以看出可执行文件类型为 Mach-O 类型，在 MAC OS 和 iOS 平台的可执行文件都是这种类型。因为我使用的是模拟器，所以处理器指令集为 x86_64。    

至此，编译过程结束。

## Mach-O文件

#### Mach-O简介
根据[官方文档](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html#//apple_ref/doc/uid/20001860-100029-TPXREF104)的描述：
  
Mach-O是OS X中二进制文件的原生可执行格式，是传送代码的首选格式。可执行格式决定了二进制文件中的代码和数据读入内存的顺序。代码和数据的顺序会影响内存使用和分页活动，从而直接影响程序的性能。 

Mach-O二进制文件被组织成段。每个部分包含一个或多个部分。段的大小由它所包含的所有部分的字节数来度量，并四舍五入到下一个虚拟内存页边界。因此，一个段总是4096字节或4千字节的倍数，其中4096字节是最小大小。   

#### Mach-O结构
Mach-O文件的结构如下：  
![Mach-O](https://user-gold-cdn.xitu.io/2018/12/18/167c0a81d53802c6?w=365&h=401&f=jpeg&s=20162)

1. Header  
	保存了Mach-O的一些基本信息，包括了平台、文件类型、LoadCommands的个数等等。
	使用```otool -v -h a.out```查看其内容： 
	 
	![Mach-o Header](https://user-gold-cdn.xitu.io/2018/12/18/167c0a684d4a0378?w=2096&h=124&f=png&s=104502)  
	
2. Load commands  
		这一段紧跟Header，加载Mach-O文件时会使用这里的数据来确定内存的分布  
3. Data  
  	包含 Load commands 中需要的各个 segment，每个 segment 中又包含多个 section。当运行一个可执行文件时，虚拟内存 (virtual memory) 系统将 segment 映射到进程的地址空间上。    
  		
  	使用```xcrun size -x -l -m a.out ```查看segment中的内容：  
  	
  	![](https://user-gold-cdn.xitu.io/2018/12/18/167c0a82a405cca6?w=1118&h=474&f=png&s=269150)
	
	- Segment __PAGEZERO。  
	大小为 4GB，规定进程地址空间的前 4GB 被映射为不可读不可写不可执行。
	- Segment __TEXT。  
	包含可执行的代码，以只读和可执行方式映射。
	- Segment __DATA。  
	包含了将会被更改的数据，以可读写和不可执行方式映射。
	- Segment __LINKEDIT。  
	包含了方法和变量的元数据，代码签名等信息。

资料：  
[Mach-O Executable Format](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html#//apple_ref/doc/uid/20001860-100029-TPXREF104)  
[编译器](https://objccn.io/issue-6-2/)  
[Mach-O 可执行文件](https://objccn.io/issue-6-3/)