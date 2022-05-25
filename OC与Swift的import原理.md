# iOS中的import，看这篇就够了！

关于iOS中的import，对于Objective-C来说，有`#import "..."`、`#import <...>`、`@import ...;`这几种方式，而对于Swift来说，只有`import ...`这一种方式。那么你了解这些import的区别吗？了解他们底层是如何查找头文件的吗？了解Objective-C文件和Swift文件是如何互相引用的吗？

## Objective-C中的头文件引用

### #include vs #import

熟悉Objective-C的都知道，在同一个target下如果我在B类中，想使用A类的API，我们首先需要在B类中将A类的头文件引入进来，那么Objective-C中有几种头文件引用的方式呢？他们的区别在于什么呢？

在Objective-C中我们可以使用的头文件引用方式有`#include`、`#import`这两种方式，`#include`做的事情其实就是简单的复制粘贴，将目标.h文件中的内容一字不落地拷贝到当前文件中，而`#import`实质上做的事情和`#include`是一样的，只不过`#import`多了一步判重逻辑，防止同一个头文件的重复引用。

我们可以创建一个简单的xcode工程来验证一下，我们创建两个Objective-C类。

![](images/OC与Swift混编/AClass.png)

![](images/OC与Swift混编/BClass.png)

然后我们在BClass的.m文件中分别使用`#include`和`#import`的方式引入AClass和BClass的头文件。然后我们使用Xcode的preprocess进行预编译的处理。

![](images/OC与Swift混编/xcodePreprocess.png)

我们可以看到结果，发现两个头文件的引用方式其实都是复制粘贴。

![](images/OC与Swift混编/BClassImportPreprocess.png)

### #import <...>

我们来修复一下AClass.h中的报错，众所周知，Objective-C的类都需要直接或者间接的继承自基类NSObject，而NSObject是属于苹果Foundation框架中的，我们引用框架中的头文件一般都是使用`#import <...>`的方式来进行引用，所以我们把AClass.h和BClass.h补充完整。

![](images/OC与Swift混编/AClassFoundation.png)

然后我们重新对BClass.m文件进行Preprocess操作看下结果。

![](images/OC与Swift混编/BCLassImportFoundationPreprocess.png)

截图中内容上除了都继承自了NSObject与之前几乎差不多，但是你会发现代码行数从45行变成了惊人的三万多行！那截图的内容前面是什么东西呢？从32268行的注释就能看出来，答案就是Foundation库中的头文件。

### 引发的问题

经过上述对#import命令的Preprocess剖析，可以发现一些早期#import所带来的问题。

第一个比较容易想到的问题就是头文件重复查找和解析的问题，首先我们几乎所有的Objective-C文件都要引入Foundation库或者UIKit库这两个大家伙，我们每个文件的头文件解析都要查找和解析这两个大家伙，会造成非常多的不必要的重复查找和解析。其次如果我们项目中有一个较底层的基础类，然后我们项目中几乎一半以上的类都要用到这个基础类，所以我们这些文件中都要引入这个基础类的头文件，那么我们这个基础类的头文件也要被重复的查找和解析多次。

还有一个不太容易发现的问题就是宏污染问题（健壮性），按上面的例子来说，如果我们在AClass.h文件中定义一个宏。

![](images/OC与Swift混编/AClassDefined.png)

当BClass引入AClass的头文件时，这个宏也会被原封不动的拷贝过来，经过xcode的预编译处理，我们看下BClass的预编译后的结果。

![](images/OC与Swift混编/BClassDefinedPreprocess.png)

可以看到BClass原本的`doSomething`的方法名被做了替换。这就是宏污染问题。被引用方的宏污染了引用方的上下文。

### PCH(PreCompiled Header)

为了改善编译的问题，Clang编译器设计了一个[PCH（Precompiled Header）](https://clang.llvm.org/docs/PCHInternals.html)的解决方案。

> The use case for precompiled headers is relatively simple: when there is a common set of headers that is included in nearly every source file in the project, we *precompile* that bundle of headers into a single precompiled header (PCH file). Then, when compiling the source files in the project, we load the PCH file first (as a prefix header), which acts as a stand-in for that bundle of headers.

正如文档所说，PCH的使用场景十分简单。当项目中几乎每个源文件中都包含一组通用头文件时，我们将该组头文件放入PCH 文件中。然后，在项目中编译源文件时，首先加载 PCH 文件，来替换这一组通用头文件。

大概原理就是，将常用的头文件放在PCH中。项目在编译时，会先解析PCH中的内容，然后缓存起来，然后在真正编译工程时再将预先编译好的产物加入到所有待编译的Source中去，这样就提高了编译的性能。

但是PCH方案会带来很多问题，比如最大的问题就是违反了迪米特原则，因为我们所有的工程内的文件都可以访问到PCH内包含的头文件的内容，所以有些文件可能会访问到完全跟自己不相关的内容，或者是不应该是自己能访问的内容，而编译器也无法准确给出错误或者警告，无形中增加了出错的可能性。

### Clang Modules

Clang在2013年推出了[Modules](https://clang.llvm.org/docs/Modules.html)的概念，Clang Modules相当于将框架进行了封装，在查找和解析框架的头文件时，只查找解析一次，并缓存在磁盘上，以便对其进行重用。

Clang Modules最大的特性之一是它不会污染上下文（context-free），也就是避免了我们上面所说的宏污染问题。 Modules的产物是被独立编译出来的，不同的Module之间是不会影响的。

那么我们如何使用Clang Modules呢？

在[Clang的Modules文档](https://clang.llvm.org/docs/Modules.html#using-modules)中明确说过，要使用Clang Modules，需要在编译器参数中加入`-fmodules`参数。

> To enable modules, pass the command-line flag `-fmodules`. This will make any modules-enabled software libraries available as modules as well as introducing any modules-specific syntax. Additional [command-line parameters](https://clang.llvm.org/docs/Modules.html#command-line-parameters) are described in a separate section later.

但是我们编译运行都依赖Xcode来进行，所以Xcode在Build Settings中提供了一个设置 - Enable Modules。

![](images/OC与Swift混编/xcodeEnableModules.png)

开启了这个选项之后，我们在编译时会自动在命令行参数中加入`-fmodules`参数开启Clang Modules。（话说这个Enable Modules的描述有些误导人，说这个设置项只会让system APIs以Clang Modules方式引入，但是这个参数是编译器参数中是否带上`-fmodules`参数的关键，所以会影响到当前target所引用的所有框架，包括Cocoapods创建的pod或者自己创建的Framework）

将它设置为Yes后，所有的通过`#import<...>`引入的系统框架都会以Clang Modules的形式引入，这个设置默认为Yes，所以不需要进行额外设置。我们来看一下设置成Yes之后，BClass.m的Preprocess产物是什么样子的。

![](images/OC与Swift混编/BClassModulesPreprocess.png)

从23行我们可以清楚地看到这里是通过Clang Modules来对Foundation框架进行import。

同时苹果在LLVM5.0引入了一个新的编译符号`@import`，使用这个符号也会告诉编译器去使用Modules方式进行引用，我们把ClassA.h中的`#import <Foundation/Foundation.h>`替换成`@import Foundation;`试一下。

![](images/OC与Swift混编/BClassModules2Preprocess.png)

可以看到编译器也没有对Foundation.h进行复制粘贴。

#### 用户框架

那对于我们自己的框架要怎么开启Clang Modules呢，不管是我们自己创建的Framework还是Cocoapods脚手架生成的库，我们都可以在对应的Target下的Build Settings中进行设置。

我们以常见的Cocoapods管理库依赖为例，我们首先创建一个PodA库，并集成进我们的主工程中。然后我们点击PodA的Build Settings，找到Packaging这一栏。

![](images/OC与Swift混编/PodAPackaging.png)

我们可以看到Defines Module这个设置项。

> If enabled, the product will be treated as defining its own module. This enables automatic production of LLVM module map files when appropriate, and allows the product to be imported as a module.

如果我们开启了这个选项，当我们使用`#import <PodA/...>`来引用时会自动以Clang Module的方式进行引入。这个效果和`#import <Foundation/Foundation>`是类似的，大家可以用Preprocess自行试验下。跟系统框架同样的，使用`@import PodA;`也是会以Clang Module的方式进行引入。我们需要注意的一点是，`@import ...;`的方式仅仅适用于开启了Defines Module的框架，如果引用不支持Clang Module的框架编译器会报Module '...' not found的错误。

如果你在Podfile中我们使用的是`use_frameworks!`，Defines Module设置项是默认为Yes，也会自动帮我们生成一个module.map文件。

通过上面的Xcode的描述可以看到开启Defines Module这个选项， 可以自动生成LLVM module map files，那么什么是module map呢？Clang编译器是怎么知道某个库是否应该以Module的方式引入呢？

#### Module Map

> The module map language describes the mapping from header files to the logical structure of modules. To enable support for using a library as a module, one must write a `module.modulemap` file for that library. The `module.modulemap` file is placed alongside the header files themselves, and is written in the module map language described below.

Clang Module是对框架的抽象描述，而module map则是这个描述的具体呈现，它对框架内的所有文件进行了结构化的描述。[Clang官方文档](https://clang.llvm.org/docs/Modules.html#module-map-language)中说到，我们要支持将框架通过Module方式引入，必须要为框架提供一个module.modulemap文件。我们看下Foundation、UIKit等系统框架的module.modulemap是什么样子的，我们在[#import <...>](####import <...>)这一节里，对BClass.m的Preprocess的截图里可以看到Foundation框架的的位置。

![](images/OC与Swift混编/FoundationLocation.png)

我们进入到Foundation.framework/Module里，看下Foundation框架的module.modulemap的内容。

![](images/OC与Swift混编/FoundationModuleMap.png)

这里描述了Foundation框架是一个framework类型的module，它的头文件时伞形头文件Foundation.h，该模块包含的任何内容都将自动重新导出（export *），同时modulemap还支持模块的嵌套定义（module ... { }），这里Foudation描述了所有的子模块包含的内容都自动导出。更多的modulemap的语法可以看下[Clang官方文档](https://clang.llvm.org/docs/Modules.html#module-map-language)。

## Xcode是如何查找头文件的

根据WWDC2018中提到的，我们可以通过Xcode的构建信息来看出一些内容。

我们随便创建一个OC项目（至少AppDelegate是OC的），在Xcode中点击编译按钮，编译结束后，我们看一下Xcode的构建日志。

![](images/OC与Swift混编/xcodeComplierM.png)

我们把这串以clang开头的命令复制到终端，并在最后加上`-v`参数运行，来显示更多的信息。通过运行结果，我们可以看到一些内容。

![](images/OC与Swift混编/terminalSearchPath.png)

通过clang编译器的输出日志，我们就可以知道Xcode是怎么查找头文件的了，他们会将双引号（"..."）的import和尖括号（<...>）的import区分开来，会分别从一个叫headermap的文件中来进行查找。那么问题来了，什么是headermap？

### HeaderMap

从WWDC视频[Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)上有提到HeaderMap

> Headermaps are used by the Xcode build system to communicate where those header files are.

我们从[clang的文档](https://clang.llvm.org/doxygen/classclang_1_1HeaderMap.html#details)中也可以看到为什么要用HeaderMap这个机制。

> To the #include file resolution process, it basically acts like a directory of symlinks to files. Its advantages are that it is dense and more efficient to create and process than a directory of symlinks.



## 参考

[WWDC2018-Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)

[Clang文档-Modules](https://clang.llvm.org/docs/Modules.html)
