# 从Clang源码深入了解Objective-C中的import

关于iOS中的import，对于Objective-C来说，有`#import "..."`、`#import <...>`、`@import ...;`这几种方式，那么你了解这些import的区别吗？了解他们底层是如何查找头文件的吗？Clang Module只能通过`#import <...>`和`@import ...;`的方式来引用吗？本文将从Clang源码的角度，详细讲解Objective-C中的头文件引用。

本文分为两部分内容，[第一部分](##Objective-C中的头文件引用)梳理一下Objective-C的头文件引用，同时介绍Header Map、Clang Modules、Module Map的概念。[第二部分](##从Clang源码深入头文件查找)通过Clang源码来深入研究Objective-C的头文件查找过程。

## Objective-C中的头文件引用

### #import "..."

熟悉Objective-C的都知道，在同一个target如果我在B类中，想使用A类的API，我们首先需要在B类中将A类的头文件引入进来，那么Objective-C中有几种头文件引用的方式呢？他们的区别在于什么呢？

在Objective-C中同一个target下我们可以使用的头文件引用方式有`#include "..."`、`#import "..."`这两种方式，它们做的事情其实就是简单的复制粘贴，将目标.h文件中的内容一字不落地拷贝到当前文件中。

我们可以创建一个简单的Xcode工程来验证一下，我们在同一个target下创建两个Objective-C类。（关于图片中的报错我们这里先忽略，后面会补充这里的内容）

![](images/OC与Swift混编/AClass.png)

![](images/OC与Swift混编/BClass.png)

我们在BClass的.m文件中分别使用`#include`和`#import`的方式引入AClass和BClass的头文件。然后我们使用Xcode的preprocess进行预编译的处理。

![](images/OC与Swift混编/xcodePreprocess.png)

我们可以看到结果，发现两个头文件的引用方式其实都是复制粘贴。

![](images/OC与Swift混编/BClassImportPreprocess.png)

而`#import`实质上做的事情和`#include`是一样的，只不过`#import`是对`#include`的一层封装，并且多了一步判重逻辑，防止同一个头文件的重复引用。

#### 从编译器Log深入头文件查找

我们深入来看下头文件的查找过程。

我们使用刚才创建的Xcode工程，我们先将工程里target的Build Settings中的`Use Header Map`选项设为No，这个选项我们稍后在介绍。然后我们进行编译。然后我们从BClass.m的编译过程来看下Clang是怎么查找BClass.h和AClass.h这两个头文件的。

我们先看一下Xcode中BClass.m的构建日志。

![](images/OC与Swift混编/xcodeComplierBClass.png)

我们将Clang的这一长串命令拷贝下来，粘贴到命令行中，然后结尾加上`-v`命令来打印更多的日志帮助我们分析。

![](images/OC与Swift混编/terminalSearchPath.png)

我们可以看到日志中`#include "..."`的路径是空的，所以Clang是有默认的查找路径的，Clang会将当前文件（BClass.m）所在的目录作为查找路径进行查找头文件，这个会在后面的Clang源码中详细讲到。所以在没有Header Map的年代，如果我们头文件在不同的文件夹目录下，引入时需要同时去指定他的文件夹。

早期的大型项目通常会用文件夹来对代码进行分类，每次引入不同文件夹的头文件需要指定所在的文件夹路径，十分繁琐。并且在文件夹中查找头文件需要访问Mac的文件夹系统来进行IO操作，在规模比较大的项目中，这种头文件查找机制的效率就会十分低下。Clang为了解决这些问题，推出了Header Map机制来优化头文件查找。

> To the #include file resolution process, it basically acts like a directory of symlinks to files. Its advantages are that it is dense and more efficient to create and process than a directory of symlinks.

#### Header Map

那么Header Map是什么东西呢？从WWDC视频[Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)上有提到HeaderMap。

> Headermaps are used by the Xcode build system to communicate where those header files are.

简单的来讲，Header Map就是将头文件和头文件的地址生成一个映射文件hmap，Clang在头文件查找时会优先读取hmap中的内容，进行头文件地址的查找。hmap本身是一个二进制文件，访问和查找效率比访问Mac文件夹系统会更高效。而开启Header Map机制只需要修改Build Settings中的`Use Header Map`选项就可，这个选项是默认开启的。

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

通过[Clang命令行参数文档](https://clang.llvm.org/docs/ClangCommandLineReference.html)可以看到-I和-iquote都是文件的搜索路径，-F是framework的搜索路径。那么-I和-iquote有什么区别呢？Clang会在查找-I、-F路径之前优先查找-iquote参数的路径，所以Clang会优先查找Xcode为我们生成的项目的hmap文件。

### #import <...>

我们再来看下`#import <...>`的头文件引用过程。

我们来修复一下最开始AClass.h中的报错，众所周知，Objective-C的类都需要直接或者间接的继承自基类NSObject，而NSObject是属于苹果Foundation框架中的，我们引用框架中的头文件一般都是使用`#import <...>`的方式来进行引用，所以我们把AClass.h和BClass.h补充完整。

![](images/OC与Swift混编/AClassFoundation.png)

然后我们重新对BClass.m文件进行Preprocess操作看下结果。

![](images/OC与Swift混编/BCLassImportFoundationPreprocess.png)

你会发现只是多了一行`#import <Foundation/Foundation.h>`，预编译后代码行数从45行变成了惊人的三万多行！从32268行的注释就能看出来，前面的代码全是Foundation库中的头文件。苹果的Framework推行的是伞形头文件，Foundation.h文件中包含了Foundation库中所有暴露出来的头文件，所以import了Foundation.h就会把整个Foundation库的头文件都import了进来，导致预编译后的内容超级多。所以将头文件简单的复制粘贴会有很多问题。

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

为了改善上述的问题，Clang编译器设计了一个[PCH（Precompiled Header）](https://clang.llvm.org/docs/PCHInternals.html)的解决方案。

> The use case for precompiled headers is relatively simple: when there is a common set of headers that is included in nearly every source file in the project, we *precompile* that bundle of headers into a single precompiled header (PCH file). Then, when compiling the source files in the project, we load the PCH file first (as a prefix header), which acts as a stand-in for that bundle of headers.

正如文档所说，PCH的使用场景十分简单。当项目中几乎每个源文件中都包含一组通用头文件时，我们将该组头文件放入PCH 文件中。然后，在项目中编译源文件时，首先加载 PCH 文件，来替换这一组通用头文件。

大概原理就是，将常用的头文件放在PCH中。项目在编译时，会先解析PCH中的内容，然后缓存起来，然后在真正编译工程时再将预先编译好的产物加入到所有待编译的Source中去，这样就提高了编译的性能。

我们在工程中创建一个pch文件，将`<Foundation/Foundation.h>`和`<PodA/PodAClassA.h>`放入pch文件内。

![](images/OC与Swift混编/pch.png)

然后在主工程中删除所有的`#import <Foundation/Foundation.h>`和`#import <PodA/PodAClassA.h> `，然后再去看下BClass.m的预编译结果。

![](images/OC与Swift混编/BClassPchPreprocess.png)

BClass.m的预编译结果又变成了简洁的30多行，同时也看不到Foundation和PodAClassA相关的字眼。

但是PCH方案也有很多问题，比如最大的问题就是违反了迪米特原则，因为我们所有的工程内的文件都可以访问到PCH内包含的头文件的内容，所以有些文件可能会访问到完全跟自己不相关的内容，或者是不应该是自己能访问的内容，而编译器也无法准确给出错误或者警告，无形中增加了出错的可能性。

#### Clang Modules

Clang在2013年推出了[Modules](https://clang.llvm.org/docs/Modules.html)的概念，Clang Modules相当于将框架进行了封装，在查找和解析框架的头文件时，只解析一次，并缓存在磁盘上，以便对其进行重用。

Clang Modules最大的特性之一是它不会被上下文污染（context-free），也就是避免了我们上面所说的宏污染问题。 Modules的产物是被独立编译出来的，不同的Module之间是不会影响的。

那么我们如何使用Clang Modules呢？

在[Clang的Modules文档](https://clang.llvm.org/docs/Modules.html#using-modules)中明确说过，要使用Clang Modules，需要在编译器参数中加入`-fmodules`参数。

> To enable modules, pass the command-line flag `-fmodules`. This will make any modules-enabled software libraries available as modules as well as introducing any modules-specific syntax. Additional [command-line parameters](https://clang.llvm.org/docs/Modules.html#command-line-parameters) are described in a separate section later.

但是我们编译运行都依赖Xcode来进行，所以Xcode在Build Settings中提供了一个设置 - Enable Modules。

![](images/OC与Swift混编/xcodeEnableModules.png)

开启了这个选项之后，我们在编译时会自动在命令行参数中加入`-fmodules`参数开启Clang Modules。（话说这个Enable Modules的描述有些误导人，说这个设置项只会让system APIs以Clang Modules方式引入，但是这个参数是编译器参数中是否带上`-fmodules`参数的关键，所以会影响到当前target的源文件所引用的所有框架，包括Cocoapods创建的Pod或者自己创建的Framework）

将它设置为Yes后，所有的通过`#import<...>`引入的系统框架都会以Clang Modules的形式引入，这个设置默认为Yes，所以不需要进行额外设置。我们开启Enable Modules后再看下BClass.m的Preprocess结果。（记得先删除掉我们刚才创建的PCH文件）

![](images/OC与Swift混编/BClassModuleFoundation.png)

从预编译结果看，通过Clang Modules进行头文件引入，并不是复制粘贴，所以就避免了宏污染的问题。而且不仅Foundation是通过Clang Module的方式引入，PodAClassA也是通过Clang Module的方式来引入的，那么对于我们自己创建的框架怎么开启Clang Module呢？

#### 用户框架

不管是我们自己创建的Framework还是Cocoapods脚手架生成的库，我们都可以在对应的Target下的Build Settings中进行设置。

如果是我们自己创建的Framework，我们首先新建一个HeaderDemoFrameworkA，在Xcode的Build Settings，找到Packaging这一栏。

![](images/OC与Swift混编/demoFrameworkPackaging.png)

我们可以看到Defines Module这个设置项。

> If enabled, the product will be treated as defining its own module. This enables automatic production of LLVM module map files when appropriate, and allows the product to be imported as a module.

如果我们开启了这个选项，当我们使用`#import <HeaderDemoFrameworkA/...>`来引用时会自动以Clang Module的方式引入。

至于使用Cocoapods管理的库，可以在Podfile中使用`use_frameworks!`或者`use_modular_headers!`来让Pod可以以Clang Module的方式进行引入。

* 如果使用的是`use_frameworks!`，Defines Module设置项默认为Yes，Cocoapods也会自动帮我们生成一个module.map文件。

* 如果使用的是`use_modular_headers!`，Defines Module设置项默认为No，Cocoapods会帮我们生成一个module.map文件，同时会在Xcode的Build Settings里的OTHER_CFLAGS和OTHER_SWIFT_FLAGS加入`-fmodule-map-file`参数，来指定module.map的路径。（Cocoapods会在project.pbxproj内每个BuildConfiguration中加入对Pods-targetName.debug/release.xcconfig的引用，`-fmodule-map-file`也是加在xcconfig文件中的）

通过上面的Xcode的描述可以看到开启Defines Module这个选项， 可以自动生成LLVM module map files，那么什么是module map呢？

#### Module Map

> The module map language describes the mapping from header files to the logical structure of modules. To enable support for using a library as a module, one must write a `module.modulemap` file for that library. The `module.modulemap` file is placed alongside the header files themselves, and is written in the module map language described below.

Clang Module是对框架的抽象描述，而module map则是这个描述的具体呈现，它对框架内的所有文件进行了结构化的描述。[Clang官方文档](https://clang.llvm.org/docs/Modules.html#module-map-language)中说到，我们要支持将框架通过Module方式引入，必须要为框架提供一个module.modulemap文件。我们看下Foundation、UIKit等系统框架的module.modulemap是什么样子的，我们在之前未开启Enable Modules时对BClass.m的Preprocess的截图里可以看到Foundation框架的的位置。

![](images/OC与Swift混编/FoundationLocation.png)

我们进入到Foundation.framework/Module里，看下Foundation框架的module.modulemap的内容。

![](images/OC与Swift混编/FoundationModuleMap.png)

这里描述了Foundation框架是一个framework类型的module，它的头文件是伞形头文件Foundation.h，该模块包含的任何内容都将自动重新导出（export *），同时modulemap还支持模块的嵌套定义（module ... { }），这里Foudation描述了所有的子模块包含的内容都自动导出。更多的modulemap的语法可以看下[Clang官方文档](https://clang.llvm.org/docs/Modules.html#module-map-language)。

### @import

苹果在LLVM5.0引入了一个新的编译符号`@import`，使用这个符号也会告诉编译器去使用Clang Modules方式来引入头文件，我们把ClassA.h中的`#import <Foundation/Foundation.h>`替换成`@import Foundation;`，看一下BClass.m的Preprocess产物是什么样子的。

![](images/OC与Swift混编/BClassModulesPreprocess.png)

从预编译结果看，通过Clang Modules进行头文件引入，并不是复制粘贴，所以就避免了宏污染的问题。

我们需要注意的一点是，`@import ...;`的方式仅仅适用于能够以Clang Module引入的框架，如果引用不支持Clang Module的框架编译器会报Module '...' not found的错误。而`#import <...>`的好处在于我们可以在Clang Module和非Module的情况下切换。

## 从Clang源码深入头文件查找

这一部分我们将从Clang源码详细探索Objective-C的头文件查找过程。在探索过程中我们会发现很多文章对头文件讲解的不严谨的地方，还会发现Xcode Preprocess功能的bug，最后会笔者会针对通过Cocoapods来管理依赖的大型项目讲解怎么对头文件查找做进一步的优化。

Clang是开源项目，所以我们通过调试Clang的源码，来看下Clang是如何查找Objective-C的头文件的。对Clang的源码调试感兴趣的，可以看下这篇[用Xcode调试Clang源码](用Xcode调试Clang源码.md)。本文中的Clang源码版本为13.0.1。

### 头文件查找路径列表

Clang要进行头文件的查找，必然需要一些路径。Xcode会通过Build Settings内的设置，在命令行中向Clang传递相关的查找路径参数。常见的参数有-I(Header Search Paths)、-F(Framework Search Paths)、-iquote(User Header Search Paths)等，通过[Clang命令行参数文档](https://clang.llvm.org/docs/ClangCommandLineReference.html)可以看到-I和-iquote都是普通头文件的搜索路径，-F是framework头文件的搜索路径。Clang通过解析这些参数，将路径存在数组中，并添加一些系统路径，最后经过排序、去重等操作生成最后的搜索路径列表SearchList。

![](images/OC与Swift混编/clangGenerateSearchList.png)

整个过程由图所示，Clang解析完命令行参数后，会将所有的路径保存在`HeaderSearchOptions.UserEntries`数组中，同时还会向数组中添加系统路径，分别是SDKROOT路径加上`/usr/local/include`和`/usr/include`，以及Clang编译器所在的目录中的include文件夹路径。

然后对解析参数后的路径列表`UserEntries`进行进一步的处理，我称之为Filter，这个步骤中，针对文件夹的路径会去过滤掉不存在的文件夹路径，针对非文件夹路径只选取hmap文件，最后再添加一个很重要的系统路径`${SDKROOT}/System/Library/Frameworks`（这个路径在不同的平台各不相同），我们使用到的几乎所有的系统framework，比如Foundation、UIKit等都存在这个目录中。将最终Filter后的结果保存在`InitHeaderSearch.IncludePath`中。

最后一步就是针对上一步的列表`IncludePath`进行排序和去重，生成最终的`SearchList`，排序的顺序是`-iquote` > `Angled`(用户传进来的-I-F等) > `system`(Clang生成的和命令行传进来的系统路径)。这也是为什么Xcode会将生成的Header Map文件使用`-iquote`传入，让同一个target下的头文件查找效率更高。

整个过程所涉及到的Clang的源码方法都已在图中标注，有兴趣的可以自行看下源码实现。

### import vs include

对于Clang来说，`#include`和`#import`这两个预编译指令，会经过词法分析（Lex）阶段拆解成token，通过命中不同的token枚举，来执行不同的方法。（至于`@import`走的不是这里的判断，后面会单独讲）

![](images/OC与Swift混编/clangLexToken.png)

我们看一下`HandleImportDirective`方法内部。

![](images/OC与Swift混编/clangHandleImportDirective.png)

我们可以看到import确实是include的一层封装。但是这层封装里并没有代码判重的逻辑，仅仅只多了语言的校验。代码判重的逻辑是在`HandleIncludeDirective`内部调用的`HandleHeaderIncludeOrImport`方法中。

![](images/OC与Swift混编/clangHandleHeaderIncludeOrImport.png)

我们可以在`HandleHeaderIncludeOrImport`方法的源码中看到，在判重逻辑之前，还进行了头文件查找和校验等一系列操作，所以虽然`#import`防止了重复引用，但是重复import还是会多次查找头文件，所以平时代码中还是尽量避免编写重复的头文件`#import`。

有的文章说`#import`是对`#include`的一层封装，封装内部添加了判重逻辑。所以这句话是不严谨的。`#import`对`#include`的封装内部其实只是多了一个Objective-C语言的校验。

### 头文件查找

我们可能会简单的认为`#import`的头文件查找就是Clang通过路径列表`SearchList`一个个的查找头文件，其实并没有我们想的这么简单。我们还可能会认为`#import "..."`与Clang Module无关，其实这也是不正确的。我们来详细看下Clang是怎么查找头文件的。

首先对于`#import "..."`方式引入的头文件，Clang会先在当前编译的.m源文件的路径中查找一次头文件，查找的方式也很简单，就是将源文件所在的文件夹目录拼上目标头文件，看下该文件是否存在。

![](images/OC与Swift混编/clangLookupFile.png)

（先在Includers中添加当前源文件的目录，如果源文件不存在会判断是否是main文件路径）

![](images/OC与Swift混编/clangLookupFileCurDir.png)

（调用`getFileAndSuggestModule`在Includer中查找头文件，`#import <...>`引入的头文件isAngled是true）

从上图的注释也可以看到，`#import <...>`引入的头文件会跳过这一步。如果当前目录没有找到的话，才会通过路径列表`SearchList`进行一个个的查找。为什么要先查找一次当前路径，我猜测是因为兼容没有启用Header Map的逻辑，因为不启用Header Map就是基于当前目录查找头文件。

查找到头文件之后，还会判断能否以Clang Module的形式引用。这里的判断与后面Normal Dir中的判断是一样的，所以会在后面讲如何判断能否以Clang Module的形式引用。

在`SearchList`查找过程中，对`#import <...>`引入的头文件查找做了特殊的处理，会跳过-iquote指定的路径。

![](images/OC与Swift混编/clangLookupAngleIndex.png)

通过判断是否是<...>（isAngled）来确定`SearchList`的初始下标从0还是AngledDirIdx开始。

![](images/OC与Swift混编/clangAngledDirIdx.png)

由`SetSearchPaths`源码可知，AngledDirIdx就是-iquote路径后面的位置。

在头文件查找路径列表`SearchList`中，Clang会将每个路径分类，一共分为三类。

* Framework Dir：常见的`-F`传入的文件夹路径。
* Normal Dir：常见的`-I、-iquote`传入的非hmap的文件夹路径。
* Header Map：hmap文件路径。

不管是哪个SearchList路径查找到了头文件，都会将该路径的下标缓存下来，如果后续有重复头文件的查找，会直接从缓存的下标路径进行查找。

#### Framework Dir

Framework Dir的头文件查找首先先会校验要查找的头文件是否是`xxx/xxx`格式。

![](images/OC与Swift混编/clangFrameworkDirNposCheck.png)

然后校验Framework是否存在。将满足FrameworkName/FileName.h格式的头文件的FrameworkName和FileName.h拆分出来，并将Dir的路径+FrameworkName+.framework/拼接成Framework的绝对路径，验证该Framework是否存在。

验证Framework存在后，开始查找FileName.h是否包含在Framework中，会分别查找Framework内的Headers和PrivateHeaders两个文件夹。

![](images/OC与Swift混编/clangLookupFrameworkHeader.png)

如果查找到了头文件就会走到最后一步判断能否以Clang Module的形式引用。首先先判断缓存中的Module是否存在，如果没有命中缓存就走正常的Module加载过程。先去查找Module Map文件，分别按顺序查找FrameworkName.framework/Modules/module.modulemap、FrameworkName.framework/module.map、FrameworkName.framework/Modules/module.private.modulemap。

![](images/OC与Swift混编/clangLookupModuleMap.png)

然后解析和加载Module Map文件，解析期间还会验证Module Map的header或者umbrella header是否包含FileName.h。最后一切顺利，就将Module Map文件路径、生成的Module等都存入缓存中，避免重复的Module解析和加载。

#### Normal Dir

Normal Dir的头文件查找分为两步，首先将查找路径拼接头文件名字组成头文件的绝对路径，然后查找头文件是否存在。

![](images/OC与Swift混编/clangNormalDirLookupFile.png)

(图中的FileName就是拼接好的绝对路径地址)

如果头文件存在，Clang还会进一步判断能否以Clang Module的形式引用。

![](images/OC与Swift混编/clangLoadNormalDirModule.png)

这里的判断能否以Clang Module的形式引用的过程，与Framework Dir中的过程有一些区别。

* Normal Dir的不会去找Module的缓存。我猜测是因为Normal Dir查找的大多都不是Framework的头文件，Framework Dir的Module缓存可以以FrameworkName作为Key缓存整个Framework的Module，整个Framework内任意的头文件查找都可以命中缓存，而Normal Dir以头文件名为Key缓存整个Module的话效率太低了，没什么意义。
* 在查找Module Map文件时，因为不是Framework路径，所以不会查找Modules/目录，只会查找Normal Dir/module.map和Normal Dir/module.modulemap这两个路径。

#### Header Map

我们来看下Clang是怎么从hmap中定位到要查找的头文件的。

![](images/OC与Swift混编/clangLookupFileHeaderMap.png)

其实就是从哈希表中查找，时间复杂度是O(1)的，效率极高。我们查看一下`HashHMapKey`方法看下这个哈希表的哈希算法。

![](images/OC与Swift混编/clangHashMapKeyHash.png)

哈希算法是将头文件每个字母的小写字母的ASCII值乘以13的累加作为哈希值。这也是我们为什么尽量避免多个头文件名称字母相同大小写不同。笔者测试了Header Map文件的创建过程中，对于相同哈希值的不同头文件，只会添加第一个到哈希表中。

如果在hmap中找到了映射的路径，这里会做一个是否是framework include的判断。如果不是framework include就查找这个路径中的文件是否存在。

![](images/OC与Swift混编/clangCheckFrameworkInclude.png)

一般来说这里framework include的判断仅针对我们在项目中手动创建的Framework。因为我们手动创建的Framework的头文件也会被写入hmap中，但是对应的地址仅仅是FrameworkName/FileName。

![](images/OC与Swift混编/terminalHmapFrameworkPath.png)

然后在hmap和后续的`SearchList`中查找FrameworkName/FileName，这种情况最后一般都会命中Framework的查找路径。这个机制其实就是Clang帮我们把`#import "FrameworkFileName.h"`自动帮我们补全成了`#import <FrameworkName/FileName.h>`，这也是`#import "..."`为什么也可以引入我们自己创建的Framework的头文件的原因。上面讲过`#import <FrameworkName/FileName.h>`是可以使用Clang Module的特性，所以这种情况使用`#import "FrameworkFileName.h"`也是可以使用Clang Module特性的。有的文章说Clang Module只能通过`#import <...>`和`@import ...;`方式来使用也是不严谨的。

### Xcode Preprocess的bug

另外值得一提的一点是，通过`#import "..."`或者通过`#import "FrameworkName/FileName.h"`引用自建的Framework的头文件，Xcode的Preprocess的显示是有bug的。我们在Demo工程中创建一个HeaderDemoFrameworkA的Framework，并在Framework内创建一个.h头文件（记得在module map的伞头文件中import该头文件，这样才可以使用Clang Module特性），并添加一个老生常谈的eat方法。

![](images/OC与Swift混编/FrameworkAClassA.png)

然后我们在老朋友BClass.m中分别通过`#import "..."`和`#import "FrameworkName/FileName.h"`的方式引入该头文件，并在引入前声明一个eat宏。

![](images/OC与Swift混编/BClassImportFramework.png)

我们通过宏污染的方式来验证FrameworkAClassA是否是通过Clang Module的方式引入的。按照头文件查找一节所讲的，`#import "FrameworkAClassA.h"`会命中Header Map，并转换成`HeaderDemoFrameworkA/FrameworkAClass.h`继续查找，最终在Framework Dir找到，并检查是否能以Clang Module方式引入。对于`#import "HeaderDemoFrameworkA/FrameworkAClass.h"`会在Framework Dir找到，并检查是否能以Clang Module方式引入。

我们编译一下，发现编译通过了，确实是通过Clang Module引入。跟我们看Clang源码知道的结果一致。但是我们Preprocess一下BClass.m。

![](images/OC与Swift混编/BClassPreprocessBug.png)

Xcode Preprocess的结果不仅不是CLang Module引入，eat方法还被宏污染了。所以Xcode Preprocess的预处理逻辑跟Clang的预处理逻辑并不是一致的！我猜测Xcode会认为`#import "..."`总会以复制粘贴的方式（非Clang Module）引入头文件。

###  @import

未完待续

### Cocoapods

未完待续

### 通过Cocoapods来管理依赖的大型项目如何优化头文件查找

未完待续

## 总结

#import封装去重

User Header Search Paths(-iquote)最先查找

import <...>会跳过User Header Search Paths

有的文章说Clang Module只能通过`#import <...>`和`@import ...;`方式来使用也是不严谨的

xcode bug

## 参考

[WWDC2018-Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)

[Clang文档-Modules](https://clang.llvm.org/docs/Modules.html)
