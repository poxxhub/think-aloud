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

### #import<...>

我们来修复一下AClass.h中的报错，众所周知，Objective-C的类都需要直接或者间接的继承自基类NSObject，而NSObject是属于苹果Foundation框架中的，我们引用框架中的头文件一般都是使用`#import<...>`的方式来进行引用，所以我们把AClass.h和BClass.h补充完整。

![](images/OC与Swift混编/AClassFoundation.png)

然后我们重新对BClass.m文件进行Preprocess操作看下结果。

![](images/OC与Swift混编/BCLassImportFoundationPreprocess.png)

截图中内容上除了都继承自了NSObject与之前几乎差不多，但是你会发现代码行数从44行变成了惊人的三万多行！那截图的内容前面是什么东西呢？答案就是Foundation库中的头文件。

### #include 和 #import 的弊端与解决方案

#### 弊端

大家可以思考一下，单纯的复制粘贴这个功能有什么弊端吗？

* 头文件重复解析问题
* 宏污染问题

第一个比较容易想到的问题就是头文件重复解析的问题，假如我们项目中有一个较底层的基础类，然后我们项目中几乎一半以上的类都要用到这个基础类，所以我们这些文件中都要引入这个基础类的头文件，那么我们就会复制粘贴N遍，那么我们这个基础类的头文件就要被重复的查找和解析N次。当然别忘了还有UIKit和Foundation这两个大家伙，我们几乎所有的类都会至少引用其中一个。

第二个问题就是宏污染问题，按上面的例子来说，如果我们在AClass的头文件中定义一个宏。

![](images/OC与Swift混编/AClassDefined.png)

当BClass引入AClass的头文件时，这个宏也会被原封不动的拷贝过来，经过xcode的预编译处理，我们看下BClass的预编译后的结果。

![](images/OC与Swift混编/BClassDefinedPreprocess.png)

可以看到BClass原本的`doSomething`的方法名被做了替换。这就是宏污染问题。被引用方的宏污染了引用方的上下文。

#### 解决方案 - PCH(PreCompiled Header)

为了改善编译的问题，clang编译器设计了一个[PCH（Precompiled Header）](https://clang.llvm.org/docs/PCHInternals.html)的解决方案。

> The use case for precompiled headers is relatively simple: when there is a common set of headers that is included in nearly every source file in the project, we *precompile* that bundle of headers into a single precompiled header (PCH file). Then, when compiling the source files in the project, we load the PCH file first (as a prefix header), which acts as a stand-in for that bundle of headers.

正如文档所说，PCH的使用场景十分简单。当项目中几乎每个源文件中都包含一组通用头文件时，我们将该组头文件放入PCH 文件中。然后，在项目中编译源文件时，首先加载 PCH 文件，来替换这一组通用头文件。

大概原理就是，将常用的头文件放在PCH中。项目在编译时，会先编译PCH中的内容，然后缓存起来，后续有其他文件引入的时候，直接加载缓存，无需再次编译。这样就提高了编译的性能。





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

https://developer.apple.com/videos/play/wwdc2018/415/
