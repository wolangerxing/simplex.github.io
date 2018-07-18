---
layout:     post
title:      "iOS 客户端启动速度优化"
subtitle:   ""
date:       2018-07-17
author:     "Simplex"
header-img: "img/post-bg-2015.jpg"
tags:
    - iOS
---

> 天下武功，唯快不破


## 前言

2017年看过一个session [启动速度优化](https://developer.apple.com/videos/play/wwdc2016/406/)，产生了优化启动速度这个想法，最终把想法落地还是有一些效果，简单总结下这块做的东西以及一些参考的资料。

PS：今年6月初的时候参加了 WWDC2018 ，Apple 号称在 iOS 12 中显著地提升了启动速度，将信将疑的更新了 iOS 12，发现的确流畅很多！不仅启动速度有优化，TableView的卡顿掉帧等现象也有所缓解（这应该和 AutoLayout 的优化有关系），只想说一句爸爸还是爸爸，辛辛苦苦做的各种优化，其实都不如Apple自己做来的效果明显。


## 正文
 
### 时间测定

冷启动和热启动差异比较大，冷启动对用户体验的影响更为关键，下面主要讲冷启动。

从用户角度来考虑，实际上真正时间的计算应该是从手机点到APP icon上，到首页展示出来的时间（或者开机广告开始展示的时间。。。）

总时间 = main()之前的加载时间 + main()之后的加载时间

下面分开说说这两块儿的影响因素以及优化方案

### main() 之前

#### 从 exec() 到 main()
- 系统内核把应用映射到新的随机地址空间(ASLR)
- 系统加载dyld
- dyld 加载 dylib：
    - load dylibs,递归:获取所依赖动态库—>找到dylib—> 调用 mmap()。
    - Rebasing:调整指向当前镜像内部资源的指针。
    - binding:调整指向镜像外部资源的指针。
- Notify ObjC Runtime
    - 注册ObjC类
    - 注册category
    - 注册selector
- Initializers
    - ObjC: +load
    - C++的构造函数属性函数 形如__attribute__((constructor)) void DoSomeInitializationWork()
    - 非基本类型的C++静态全局变量的创建(通常是类或结构体)
- dyld Calls main() 

上面每一步展开都有更深的东西可以说，提供几个不错的参考资料，有兴趣的可以进行更人的研究~

[官方讲解](https://developer.apple.com/videos/play/wwdc2016/406/)

[今日头条启动速度优化](https://techblog.toutiao.com/2017/01/17/iosspeed/)

[iOS 程序 main 函数之前发生了什么](https://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

[dyld与ObjC](https://blog.cnbluebox.com/blog/2017/06/20/dyldyu-objc/)

#### 优化策略

总的来说，main() 之前干的事情其实就是把加载动态库，向runtime中注册类和方法列表，一些初始化方法执行，因此我们的策略如下

- 检查pod中不必要的framework并删除，链接动态库的时间会占据 main() 之前大部分的时间
- 可以使用类似AppCode代码检查功能的工具清理项目中的无用的类
- 删减无用的方法 [官方](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/CheckingCodeCoverage.html) [一个方法](https://stackoverflow.com/questions/35233564/how-to-find-unused-code-in-xcode-7#comment58182394_35233564)
- 删减无用的静态变量 
- 将在 +load() 做的事情放到 +initialize() 中，这里还是有一些需要注意的细节，参考 [+load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)
- 尽量不要用C++虚函数(创建虚函数表有开销)

### main() 之后

这部分时间的大部分跟业务逻辑有关系。

首先，我们可以通过打印每个方法执行的时间来判断出哪些是对首屏显示有较大影响的？这个业务逻辑是否一定要放在启动的时候？能否完全异步或者延后一小段时间处理？

其次我们可以适当减少或延迟在首页的 controller 的 viewDidLoad() 和 viewWillAppear() 中方法调用，这两个方法的耗时还是比较多的。

### 总结

上述事情做完后，优化的效果还是比较明显的，大致提升了30%左右，当然还是不如苹果爸爸直接在 iOS 12 中把整体的启动速度提升40%， 完。

---
