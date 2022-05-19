# Objective-C与Swift的混编原理

## Objective-C中的头文件引用

### #include vs #import

熟悉Objective-C的都知道，如果我在B类中，想使用A类的API，我们首先需要在B类中将A类的头文件引入进来，那么Objective-C中有几种头文件引用的方式呢？他们的区别在于什么呢？

在Objective-C中我们可以使用的头文件引用方式有`include`、`import`这两种方式，它们的实现原理是什么呢?

众所周知，`#include`、`#import`都是系统提供的两个预编译指令，所以我们可以用xcode提供的预编译功能来看一下这两个指令都做了什么。

首先我们创建一个xcode工程，然后我们创建两个Objective-C类。

![](images/OC与Swift混编/AClass.png)

![](images/OC与Swift混编/BClass.png)

然后我们在BClass的.m文件中分别使用`#include`和`#import`的方式引入AClass的头文件。然后我们使用xcode的preprocess进行预编译的处理。

![](images/OC与Swift混编/xcodePreprocess.png)

我们可以看到结果，发现两个头文件的引用方式完全一样。

![](images/OC与Swift混编/BClassImportPreprocess.png)

他们做的事情就是简单的复制粘贴！将目标.h的文件的内容完全拷贝到当前文件中，并替换掉这句头文件引用。那么他们到底有什么区别呢，他们区别就是`#import`是`#include`预编译指令的微小创新，`#import`除了对目前头文件的复制粘贴之外，还添加了一个避免头文件重复引用的功能。大家可以自己分别使用`#include`和`#import`两次AClass的头文件，再使用preprocess功能查看下区别。

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

我们从clang的文档中

## 参考

https://developer.apple.com/videos/play/wwdc2018/415/
