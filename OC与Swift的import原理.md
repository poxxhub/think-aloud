# iOS中的import，看这篇就够了！

关于iOS中的import，对于Objective-C来说，有`#import "..."`、`#import <...>`、`@import ...;`这几种方式，而对于Swift来说，只有`import ...`这一种方式。那么你了解这些import的区别吗？了解他们底层是如何查找头文件的吗？了解Objective-C文件和Swift文件是如何互相引用的吗？本文将从Clang源码的角度，详细讲解头文件引用。

## Objective-C中的头文件引用

### #import "..."

熟悉Objective-C的都知道，在同一个target如果我在B类中，想使用A类的API，我们首先需要在B类中将A类的头文件引入进来，那么Objective-C中有几种头文件引用的方式呢？他们的区别在于什么呢？

在Objective-C中我们可以使用的头文件引用方式有`#include "..."`、`#import "..."`这两种方式，它们做的事情其实就是简单的复制粘贴，将目标.h文件中的内容一字不落地拷贝到当前文件中。

我们可以创建一个简单的Xcode工程来验证一下，我们在同一个target下创建两个Objective-C类。（关于图片中的报错我们这里先忽略，后面会补充这里的内容）

![](images/OC与Swift混编/AClass.png)

![](images/OC与Swift混编/BClass.png)

我们在BClass的.m文件中分别使用`#include`和`#import`的方式引入AClass和BClass的头文件。然后我们使用Xcode的preprocess进行预编译的处理。

![](images/OC与Swift混编/xcodePreprocess.png)

我们可以看到结果，发现两个头文件的引用方式其实都是复制粘贴。

![](images/OC与Swift混编/BClassImportPreprocess.png)

而`#import`实质上做的事情和`#include`是一样的，只不过`#import`是对`#include`的一层封装，并且多了一步判重逻辑，防止同一个头文件的重复引用。

#### 从Clang源码看#import "..."引用过程

接下来从我们从Clang源码中看一下同一个target下的头文件是怎么引用的。



后面`HandleHeaderIncludeOrImport`方法会进行头文件的查找，查找到头文件之后，预编译器会调用`EnterSourceFile`方法进行复制粘贴。

![](images/OC与Swift混编/clangEnterSourceFile.png)

#### 深入头文件查找

我们深入来看下头文件的查找过程。

我们使用刚才创建的Xcode工程，我们先将工程里target的Build Settings中的`Use Header Map`选项设为No，这个选项我们稍后在介绍。然后我们进行编译。然后我们从BClass.m的编译过程来看下Clang是怎么查找BClass.h和AClass.h这两个头文件的。

我们先看一下Xcode中BClass.m的构建日志。

![](images/OC与Swift混编/xcodeComplierBClass.png)

我们将Clang的这一长串命令拷贝下来，粘贴到命令行中，然后结尾加上`-v`命令来打印更多的日志帮助我们分析。

![](images/OC与Swift混编/terminalSearchPath.png)

我们可以看到日志中`#include "..."`的路径是空的，所以我们猜测Clang是有默认的查找路径的，我们调试一下Clang的源码看下内部实现。我们在预编译器的查找文件方法（Preprocessor::LookupFile）中看到，确实会将当前文件（BClass.m）所在的目录作为查找路径进行查找头文件。所以在没有Header Map的年代，如果我们头文件在不同的文件夹目录下，引入时需要同时去指定他的文件夹。

![](images/OC与Swift混编/clangLookupFile.png)

早期的大型项目通常会用文件夹来对代码进行分类，每次引入不同文件夹的头文件需要指定所在的文件夹路径，十分繁琐。并且在文件夹中查找头文件需要访问Mac的文件夹系统来进行操作，在规模比较大的项目中，这种头文件查找机制的效率就会十分低下。Clang为了解决这些问题，在某个版本中推出了Header Map机制来优化头文件查找。

> To the #include file resolution process, it basically acts like a directory of symlinks to files. Its advantages are that it is dense and more efficient to create and process than a directory of symlinks.

#### Header Map

那么Header Map是什么东西呢？从WWDC视频[Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)上有提到HeaderMap。

> Headermaps are used by the Xcode build system to communicate where those header files are.

简单的来讲，Header Map就是将头文件和头文件的地址生成一个映射文件hmap，Clang在头文件查找时会优先读取hmap中的内容，进行头文件地址的查找。hmap本身是一个二进制文件，访问和查找效率比访问Mac文件夹系统会更高效。而开启Header Map机制只需要修改Build Settings中的`Use Header Map`选项就可，这个机制是默认开启的。

我们将Xcode工程里target的Build Settings中的`Use Header Map`选项改回为Yes，重新编译一下。我们看一下构建日志，会发现Xcode在编译源文件之前会提前生成hmap文件。

![](images/OC与Swift混编/xcodeWriteHeaderMap.png)

我们还是将BClass.h的Clang命令加上`-v`粘贴到命令行中。

![](images/OC与Swift混编/terminalHmapSearchPath.png)

可以看到`#include "..."`的路径变成了两个hmap文件，我们使用[hmap命令行工具](https://github.com/milend/hmap)，来查看下这两个hmap文件。

![](images/OC与Swift混编/terminalPrintDotSearchPath.png)

可以看到`HeaderDemo-project-headers.hmap`文件是target中所有头文件的及其路径的映射表。

笔者没有找到hmap的具体生成逻辑的源码（有大佬知道的话麻烦联系下我~），我们可以根据结果来猜测，Xcode会将target目录下的所有.h文件及其地址生成hmap，那Xcode从哪里存的.h文件的地址呢？答案是在工程文件xcodeproj内project.pbxproj文件中的`PBXFileReference section`存了所有源文件的地址，我们在target内添加或者删除文件，修改的也是这个project.pbxproj文件。所以Xcode会将`PBXFileReference section`里的所有.h头文件及其地址生成映射写入hmap中。大家可以删除`PBXFileReference section`内的指定头文件然后重新编译和打印hmap来验证这个猜想。

我们知道了Header Map就是头文件和地址的映射之后，我们看一下Clang是怎么通过Header Map查找头文件的。首先我们将上面的Clang命令格式化成更易读的格式。

![](images/OC与Swift混编/terminalClangDemo.png)

通过[Clang命令行参数文档](https://clang.llvm.org/docs/ClangCommandLineReference.html)可以看到-I和-iquote都是文件的搜索路径，-F是framework的搜索路径，所以我们传递给Clang九个搜索路径。那么-I和-iquote有什么区别呢？

![](images/OC与Swift混编/clangArgsQuoted.png)

从Clang源码可以看到-iquote参数的路径是放在最前面进行查找，所以看起来Clang会优先查找Xcode为我们生成的项目的hmap文件。别急，事实还真的不是这样。Clang确实会优先查找hmap文件，但是在查找hmap文件之前，Clang还做了额外的查找。Clang会将要查找的头文件拼上当前编译的.m文件的路径，先查找一次头文件是否是在当前路径下。

![](images/OC与Swift混编/clangLookupFile.png)

（先在Includers中添加当前目录）

![](images/OC与Swift混编/clangLookupFileCurDir.png)

（调用`getFileAndSuggestModule`在Includer中查找头文件）

为什么要先查找一次当前路径，我猜测原因有两点：

1. 兼容没有启用Header Map的逻辑，因为不启用Header Map就是基于当前目录查找头文件。
2. 如果大型项目中，hmap的头文件映射会非常多，从hmap中定位要查找的头文件是O(n)的时间复杂度，也需要一定时间。如果能通过头文件的绝对路径先去查找一下，会节省从hmap定位头文件的时间。

如果在当前目录没有找到头文件，才会进行从搜索路径列表中开始查找。

![](images/OC与Swift混编/clangLookupFileSearchPath.png)

还记得之前Clang把hmap放在了搜索路径列表中的最前面了吗，所以Clang将会从hmap开始进行搜索。我们来看下Clang是怎么从hmap中定位到要查找的头文件的。

![](images/OC与Swift混编/clangLookupFileHeaderMap.png)

虽然注释中说是从哈希表中查找，但是从代码中看到却是通过迭代了哈希表中所有的key，然后匹配到之后再从哈希表中拿到地址，所以时间复杂度不是O(1)而是O(n)的。

如果hmap找不到，就会进行-I参数的路径查找，从MacOS的文件系统中查找头文件。

### #import <...>

我们再来看下`#import <...>`的头文件引用过程。

我们来修复一下最开始AClass.h中的报错，众所周知，Objective-C的类都需要直接或者间接的继承自基类NSObject，而NSObject是属于苹果Foundation框架中的，我们引用框架中的头文件一般都是使用`#import <...>`的方式来进行引用，所以我们把AClass.h和BClass.h补充完整。

![](images/OC与Swift混编/AClassFoundation.png)

然后我们重新对BClass.m文件进行Preprocess操作看下结果。

![](images/OC与Swift混编/BCLassImportFoundationPreprocess.png)

截图中内容上除了都继承自了NSObject与之前几乎差不多，但是你会发现代码行数从45行变成了惊人的三万多行！那截图的内容前面是什么东西呢？从32268行的注释就能看出来，答案就是Foundation库中的头文件。苹果的framework推行的是伞形头文件，Foundation.h文件中包含了Foundation库中所有暴露出来的头文件，所以import了Foundation.h就会把整个Foundation库的头文件都import了进来，导致预编译后的内容超级多。

#### 早期#import <...>引发的问题

经过上述对#import命令的Preprocess剖析，可以发现一些早期`#import <...>`引入框架头文件所带来的问题。

第一个比较容易想到的问题就是头文件重复查找和解析的问题，首先我们几乎所有的Objective-C文件都要引入Foundation库或者UIKit库这两个大家伙，我们每个文件的头文件都要查找和解析这两个大家伙，会造成非常多的不必要的重复查找和解析。其次如果我们项目中有一个较底层的基础库（比如Masonry），然后我们项目中几乎一半以上的类都要用到这个基础库，所以我们这个基础库的头文件也要被重复的查找和解析多次。

还有一个不太容易发现的问题就是宏污染问题（健壮性）。按上面的例子来说，我们创建一个PodA的库并集成进我们的主工程中（这里使用Cocoapods来创建和集成），然后再PodA库中添加一个Objective-C的类PodAClassA，在PodAClassA.h中声明一个eat方法。

![](images/OC与Swift混编/PodAClassA.png)

BClass.m中通过`#import <PodA/PodAClassA.h>`引入头文件，并且在最上方定义一个eat的宏。

![](images/OC与Swift混编/BClassDefined.png)

经过xcode的预编译处理，我们看下BClass的预编译后的结果。

![](images/OC与Swift混编/BClassDefinedPreprocess.png)

可以看到BClass.m中doSomething内的eat被做了替换，这是我们预期中的，但是我们发现PodAClassA.h中声明的eat也被做了替换。这就是宏污染问题，也就是当前文件的宏污染了引用进来的头文件。

#### PCH(PreCompiled Header)

为了改善编译的问题，Clang编译器设计了一个[PCH（Precompiled Header）](https://clang.llvm.org/docs/PCHInternals.html)的解决方案。

> The use case for precompiled headers is relatively simple: when there is a common set of headers that is included in nearly every source file in the project, we *precompile* that bundle of headers into a single precompiled header (PCH file). Then, when compiling the source files in the project, we load the PCH file first (as a prefix header), which acts as a stand-in for that bundle of headers.

正如文档所说，PCH的使用场景十分简单。当项目中几乎每个源文件中都包含一组通用头文件时，我们将该组头文件放入PCH 文件中。然后，在项目中编译源文件时，首先加载 PCH 文件，来替换这一组通用头文件。

大概原理就是，将常用的头文件放在PCH中。项目在编译时，会先解析PCH中的内容，然后缓存起来，然后在真正编译工程时再将预先编译好的产物加入到所有待编译的Source中去，这样就提高了编译的性能。

我们在工程中创建一个pch文件，将`<Foundation/Foundation.h>`和`<PodA/PodAClassA.h>`放入pch文件内。

![](images/OC与Swift混编/pch.png)

然后在主工程中删除所有的`#import <Foundation/Foundation.h>`和`#import <PodA/PodAClassA.h> `，然后再去看下BClass.m的预编译结果。

![](images/OC与Swift混编/BClassPchPreprocess.png)

BClass.m的预编译结果又变成了简洁的30多行，同时也看不到Foundation和PodAClassA相关的字眼。

PCH方案也有很多问题，比如最大的问题就是违反了迪米特原则，因为我们所有的工程内的文件都可以访问到PCH内包含的头文件的内容，所以有些文件可能会访问到完全跟自己不相关的内容，或者是不应该是自己能访问的内容，而编译器也无法准确给出错误或者警告，无形中增加了出错的可能性。

#### Clang Modules

Clang在2013年推出了[Modules](https://clang.llvm.org/docs/Modules.html)的概念，Clang Modules相当于将框架进行了封装，在查找和解析框架的头文件时，只查找解析一次，并缓存在磁盘上，以便对其进行重用。

Clang Modules最大的特性之一是它不会被上下文污染（context-free），也就是避免了我们上面所说的宏污染问题。 Modules的产物是被独立编译出来的，不同的Module之间是不会影响的。

那么我们如何使用Clang Modules呢？

在[Clang的Modules文档](https://clang.llvm.org/docs/Modules.html#using-modules)中明确说过，要使用Clang Modules，需要在编译器参数中加入`-fmodules`参数。

> To enable modules, pass the command-line flag `-fmodules`. This will make any modules-enabled software libraries available as modules as well as introducing any modules-specific syntax. Additional [command-line parameters](https://clang.llvm.org/docs/Modules.html#command-line-parameters) are described in a separate section later.

但是我们编译运行都依赖Xcode来进行，所以Xcode在Build Settings中提供了一个设置 - Enable Modules。

![](images/OC与Swift混编/xcodeEnableModules.png)

开启了这个选项之后，我们在编译时会自动在命令行参数中加入`-fmodules`参数开启Clang Modules。（话说这个Enable Modules的描述有些误导人，说这个设置项只会让system APIs以Clang Modules方式引入，但是这个参数是编译器参数中是否带上`-fmodules`参数的关键，所以会影响到当前target的源文件所引用的所有框架，包括Cocoapods创建的pod或者自己创建的Framework）

将它设置为Yes后，所有的通过`#import<...>`引入的系统框架都会以Clang Modules的形式引入，这个设置默认为Yes，所以不需要进行额外设置。同时苹果在LLVM5.0引入了一个新的编译符号`@import`，使用这个符号也会告诉编译器去使用Modules方式进行引用，我们把ClassA.h中的`#import <Foundation/Foundation.h>`替换成`@import Foundation;`，看一下BClass.m的Preprocess产物是什么样子的。（记得先删除掉我们刚才创建的PCH文件）

![](images/OC与Swift混编/BClassModulesPreprocess.png)

从预编译结果看，通过Clang Modules进行头文件引入，并不是复制粘贴，所以就避免了宏污染的问题。

#### 用户框架

那对于我们自己的框架要怎么开启Clang Modules呢，不管是我们自己创建的Framework还是Cocoapods脚手架生成的库，我们都可以在对应的Target下的Build Settings中进行设置。

如果是我们自己创建的Framework，我们首先新建一个HeaderDemoFrameworkA，在Xcode的Build Settings，找到Packaging这一栏。

![](images/OC与Swift混编/demoFrameworkPackaging.png)

我们可以看到Defines Module这个设置项。

> If enabled, the product will be treated as defining its own module. This enables automatic production of LLVM module map files when appropriate, and allows the product to be imported as a module.

如果我们开启了这个选项，当我们使用`#import <HeaderDemoFrameworkA/...>`或者`@import HeaderDemoFrameworkA;`来引用时会自动以Clang Module的方式引入。我们需要注意的一点是，`@import ...;`的方式仅仅适用于能够以Clang Module引入的框架，如果引用不支持Clang Module的框架编译器会报Module '...' not found的错误。而`#import <...>`的好处在于我们可以在Clang Module和非Module的情况下切换。

至于使用Cocoapods管理的库，可以在Podfile中使用`use_frameworks!`或者`use_modular_headers!`来让Pod可以以Clang Module的方式进行引入。

* 如果使用的是`use_frameworks!`，Defines Module设置项默认为Yes，Cocoapods也会自动帮我们生成一个module.map文件。

* 如果使用的是`use_modular_headers!`，Defines Module设置项默认为No，Cocoapods会帮我们生成一个module.map文件，同时会在Xcode的Build Settings里的OTHER_CFLAGS和OTHER_SWIFT_FLAGS加入`-fmodule-map-file`参数，来指定module.map的路径。（Cocoapods会在project.pbxproj内每个BuildConfiguration中加入对Pods-targetName.debug/release.xcconfig的引用，`-fmodule-map-file`也是加在xcconfig文件中的）

通过上面的Xcode的描述可以看到开启Defines Module这个选项， 可以自动生成LLVM module map files，那么什么是module map呢？

#### Module Map

> The module map language describes the mapping from header files to the logical structure of modules. To enable support for using a library as a module, one must write a `module.modulemap` file for that library. The `module.modulemap` file is placed alongside the header files themselves, and is written in the module map language described below.

Clang Module是对框架的抽象描述，而module map则是这个描述的具体呈现，它对框架内的所有文件进行了结构化的描述。[Clang官方文档](https://clang.llvm.org/docs/Modules.html#module-map-language)中说到，我们要支持将框架通过Module方式引入，必须要为框架提供一个module.modulemap文件。我们看下Foundation、UIKit等系统框架的module.modulemap是什么样子的，我们在[#import <...>](####import <...>)这一节里，对BClass.m的Preprocess的截图里可以看到Foundation框架的的位置。

![](images/OC与Swift混编/FoundationLocation.png)

我们进入到Foundation.framework/Module里，看下Foundation框架的module.modulemap的内容。

![](images/OC与Swift混编/FoundationModuleMap.png)

这里描述了Foundation框架是一个framework类型的module，它的头文件是伞形头文件Foundation.h，该模块包含的任何内容都将自动重新导出（export *），同时modulemap还支持模块的嵌套定义（module ... { }），这里Foudation描述了所有的子模块包含的内容都自动导出。更多的modulemap的语法可以看下[Clang官方文档](https://clang.llvm.org/docs/Modules.html#module-map-language)。

#### 从Clang源码看#import <...>引用过程

Clang编译器是怎么知道某个库是否应该以Module的方式引入呢？

## 从Clang源码深入头文件查找

Clang是开源项目，所以我们通过调试Clang的源码，来看下Clang是如何查找Objective-C的头文件的。对Clang的源码调试感兴趣的，可以看下这篇[用Xcode调试Clang源码](用Xcode调试Clang源码.md)。本文中的Clang源码版本为13.0.1。

### 头文件查找路径列表

Clang要进行头文件的查找，必然需要一些路径。Xcode会通过Build Settings内的设置，在命令行中向Clang传递相关的查找路径参数。常见的参数有-I(Header Search Paths)、-F(Framework Search Paths)、-iquote(User Header Search Paths)等，通过[Clang命令行参数文档](https://clang.llvm.org/docs/ClangCommandLineReference.html)可以看到-I和-iquote都是普通头文件的搜索路径，-F是framework头文件的搜索路径。Clang通过解析这些参数，将路径存在数组中，并添加一些系统路径，最后经过排序、去重等操作生成最后的搜索路径列表SearchList。

![](images/OC与Swift混编/clangGenerateSearchList.png)

整个过程由图所示，Clang解析完命令行参数后，会将所有的路径保存在`UserEntries`数组中，同时还会向数组中添加系统路径，分别是SDKROOT路径加上`/usr/local/include`和`/usr/include`，以及Clang编译器所在的目录中的include文件夹路径。

拿到解析参数后的路径列表进行进一步的处理，我称之为Filter，这个步骤中，会针对文件夹的路径去过滤掉不存在的文件夹路径，针对非文件夹路径只选取hmap文件，最后再添加一个很重要的系统路径`${SDKROOT}/System/Library/Frameworks`（这个路径在不同的平台各不相同），我们使用到的几乎所有的系统framework，比如Foundation、UIKit等都存在这个目录中。

最后一步就是针对上一步的列表进行排序和去重，排序的顺序是`-iquote` > `Angled`(用户传进来的-I-F等) > `system`(Clang生成的系统路径)。这也是为什么Xcode会将生成的Header Map文件使用`-quote`传入，让开发者定义的同一个target下的头文件查找效率更高。

整个过程所涉及到的Clang的源码方法都已在图中标注，有兴趣的可以自行看下源码实现。

### import vs include

对于Clang来说，`#include`和`#import`这两个预编译指令，会经过词法分析（Lex）阶段拆解成token，通过命中不同的token枚举，来执行不同的方法。

![](images/OC与Swift混编/clangLexToken.png)

我们看一下`HandleImportDirective`方法内部。

![](images/OC与Swift混编/clangHandleImportDirective.png)

我们可以看到import确实是include的一层封装。但是这层封装里并没有代码判重的逻辑，仅仅只多了语言的校验。代码判重的逻辑是在`HandleIncludeDirective`内部调用的`HandleHeaderIncludeOrImport`方法中。

![](images/OC与Swift混编/clangHandleHeaderIncludeOrImport.png)

我们可以在`HandleHeaderIncludeOrImport`方法的源码中看到，在判重逻辑之前，还进行了头文件查找和校验等一系列操作，所以虽然`#import`防止了多次复制，但是重复import还是会多次查找头文件，所以平时代码中还是尽量避免编写重复的头文件`#import`。

有的文章说`#import`是对`#include`的一层封装，封装内部添加了判重逻辑。所以这句话是不严谨的。`#import`对`#include`的封装内部其实只是多了一个Objective-C语言的校验。

### #import "..."

`#import "..."`一般有以下使用场景：

1. 引用同一个target内部的头文件。例如#import "FileA.h"
2. 引用自己创建的Framework内的头文件。例如#import "FrameworkName/FileB.h"或者#import "FileB.h"
3. 引用Cocoapods并且使用use_modular_headers配置创建的Pod内的头文件。例如#import "PodName/FileC.h"或者#import "FileC.h"

我们可能会简单的认为`#import "..."`的头文件查找就是Clang通过路径列表SearchList一个个的查找头文件，其实并没有我们想的这么简单。

我把Clang通过`#import "..."`方式引用的头文件的查找过程，做了简单的流程图。

![](images/OC与Swift混编/clangSearchHeader.png)

首先Clang会先在当前编译的.m源文件的路径中查找一次头文件，查找的方式也很简单，就是将源文件所在的文件夹目录拼上目标头文件，看下该文件是否存在。如果当前目录没有找到的话，才会通过路径列表SearchList进行一个个的查找。

为什么要先查找一次当前路径，我猜测是因为兼容没有启用Header Map的逻辑，因为不启用Header Map就是基于当前目录查找头文件。

还记得Clang生成路径列表SearchList时把hmap放在了最前面了吗，所以Clang将会从hmap开始进行搜索。我们来看下Clang是怎么从hmap中定位到要查找的头文件的。

![](images/OC与Swift混编/clangLookupFileHeaderMap.png)

其实就是从哈希表中查找，时间复杂度是O(1)的，效率极高。我们查看一下HashHMapKey方法看下这个哈希表的哈希算法。

![](images/OC与Swift混编/clangHashMapKeyHash.png)

哈希算法使用的是头文件每个字母的小写字母的ASCII值乘以13的累加作为哈希值。这也是我们为什么尽量避免多个头文件名称字母相同大小写不同。笔者测试了Header Map文件的创建过程中，对于相同哈希值的不同头文件，只会添加第一个到哈希表中。

如果在hmap中找到了映射的路径，这里会做一个是否是framework include的判断。如果不是framework include就查找这个路径中的文件是否存在。

![](images/OC与Swift混编/clangCheckFrameworkInclude.png)

这里framework include的判断仅仅是针对我们在项目中手动创建的Framework，在使用场景中提到的#import "FileB.h"就会命中这种情况。因为我们手动创建的Framework的头文件也会被写入hmap中，但是对应的地址仅仅是FrameworkName/FileName。

![](images/OC与Swift混编/terminalHmapFrameworkPath.png)

然后在hmap和后续的SearchList中查找FrameworkName/FileName，这种情况最后一般都会命中Framework的查找路径。这个机制其实就是Clang帮我们把`#import "FrameworkFileName.h"`自动帮我们补全成了`#import <FrameworkName/FileName.h>`，这也是`#import "..."`为什么也可以引入我们自己创建的Framework的头文件的原因。上面讲过`#import <FrameworkName/FileName.h>`是可以使用Clang Module的特性，所以这种情况使用`#import "FrameworkFileName.h"`也是可以使用Clang Module特性的。有的文章说Clang Module只能通过`#import <...>`和`@import ...;`方式来使用也是不严谨的。

还有另外值得一提的一点是关于使用场景中的第二条，通过`#import "..."`引用自建的Framework的头文件，Xcode的Preprocess的显示是有bug的。我们在Demo工程中创建一个HeaderDemoFrameworkA的Framework，并在Framework内创建一个.h头文件（记得在module map的伞头文件中import该头文件，这样才可以使用Clang Module特性），并添加一个老生常谈的eat方法。

![](images/OC与Swift混编/FrameworkAClassA.png)

然后我们在老朋友BClass.m中通过`#import "..."`的方式引入该头文件，并在引入前声明一个eat宏。

![](images/OC与Swift混编/BClassImportFramework.png)

我们通过宏污染的方式来验证FrameworkAClassA是否是通过Clang Module的方式引入的。我们编译一下，发现编译通过了，确实是通过Clang Module引入。跟我们看Clang源码知道的结果一致。但是我们Preprocess一下BClass.m。

![](images/OC与Swift混编/BClassPreprocessBug.png)

Xcode Preprocess的结果不仅不是CLang Module引入，eat方法还被宏污染了。我们猜测Xcode Preprocess的预处理逻辑也许跟Clang的预处理逻辑并不是一致的。

回到hmap的查找过程中，如果在hmap中没有找到要查找的目标头文件，就会在文件夹目录中进行查找，文件夹路径分为Framework文件夹路径（常见-F参数传入）和非Framework文件夹路径。Xcode在给Clang传参的时候，一般都会把-F参数放在最后，所以正常情况下，Clang会先从非Framework文件夹路径中查找头文件。

在从非Framework文件夹路径中查找其实就是简单的路径拼接头文件名字组成头文件的绝对路径，然后查找头文件是否存在。如果头文件存在，Clang还会进一步判断能否以Clang Module的形式引用。先查找Module Map文件是否存在，检查头文件所在的文件夹是否是.framework结尾（正常情况下我们使用#import "..."的方式引用时都不是.framework结尾）。如果是framework，则查找Modules/module.modulemap、module.map和Modules/module.private.modulemap是否存在，如果不是framework，则查找module.modulemap和module.map是否存在。找到Module Map文件之后再去加载解析Module Map。

在Framework文件夹中查找时，会对头文件做了校验，仅支持`#import "xxx/xxx.h"`的格式，这种格式我们平常开发中很少会用到。对于详细的查找过程，与`#import <...>`一致，我们在下一节进行讲解。

Cocoapods回忆PodName.modulemap

### #import <...>



## 总结

#import封装去重

User Header Search Paths(-iquote)最先查找

import <...>会跳过User Header Search Paths

有的文章说Clang Module只能通过`#import <...>`和`@import ...;`方式来使用也是不严谨的

xcode bug

## 参考

[WWDC2018-Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)

[Clang文档-Modules](https://clang.llvm.org/docs/Modules.html)
