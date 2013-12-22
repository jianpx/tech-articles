CocoaPods 初阶
==============
* 什么是CocoaPods？它的作用是什么？

  =》[CocoaPods](http://cocoapods.org)是一个第三方资源管理工具，这里的"资源"可以是Objective C的源码，也可以是图片集合（bundle)，不过除了管理这些资源以外， 它还管理这些“资源”之间的依赖。所以总的来说，CocoaPods是资源+依赖的管理器工具。平常大家都喜欢把第三方资源说成包，下面也沿用这样的说法。或者打个比喻把，CocoaPods的角色其实跟python里面的pip很像。在没有CocoaPods之前，我们要使用第三方（别人）的包的时候需要手动添加源码、各种依赖的framework，有些还要添加编译参数，这是何等的麻烦以及容易出错，而且不自动化，就像旧石器时代一样，但是CocoaPods的出现象征着这一切都可以自动化、便利的使用和管理，一句`pod install` 就解决问题。所以CocoaPods这个项目在[github](https://github.com/CocoaPods/CocoaPods)才如此受欢迎！！！

* 安装、使用[主要针对开发者的使用]

  1. *安装*: 先安装ruby和Xcode command line tools 然后在终端执行`[sudo] gem install cocoapods` 之后 `pod setup` 即可。如果遇到`cocoaPods pod install Permission denied`权限问题（之前用Mac OSX 10.7时遇过，之后用10.8就没有了），可以参考stackoverflow的[cocoaPods pod install Permission denied](http://stackoverflow.com/questions/16049335/cocoapods-pod-install-permission-denied/17542841#17542841)
  2. *使用*: 在Xcode的XXX.xcodeproj 文件所在的目录创建一个Podfile的文件, 编辑里面的内容如下：

    platform :ios, '5.0'

    pod 'AFNetworking', '~> 1.3.2'

  保存之后使用`pod install` 命令就会安装第三方包AFNetworking（网络处理）并且以子项目的形式集成到你的项目中，从此之后你就要打开<your_project_name>.xcworkspace 来开发了。就这么简单，有点像`apt-get  install xxx`一样

CocoaPods 进阶
=============
* 原理：
  * *主要文件*：请看下图![cocoapods组成](https://github.com/jianpx/tech-articles/raw/master/images/CocoaPods%20components.png)
  * *pod install* 做了什么：请看下图：
    ![pod install inner](https://github.com/jianpx/tech-articles/raw/master/images/PodInstall.png)

    整个流程下来，你的项目中就会多了<target_name>.xcworkspace 文件、Pods目录、Podfile.lock文件，Podfile（之前就有）。而且你打开xcworkspace文件会发现，在你项目的Framework里面多了一个`libPods.a`的文件，这个二进制包就是cocoapod产生的，里面整合了你所有第三方包的东西，你就能任意使用这些第三方包了。再多讲一点，你项目的Target里面的Build Phases的`Link Binary With Libraries` 也会包含了这个`libPods.a`文件。如果第三方包有bundle之类的资源，cocoapod会通过`"${SRCROOT}/Pods/Pods-resources.sh"`这个脚本将资源copy到对应位置。了解上述流程对下面的内容是有帮助的。

* 如果某一天你也开发了个库或者是组件想给别人用，也希望别人只要`pod install`就能快速安装使用，该怎么办?下面就来教你。假设你的组件叫RestUI
  * => 在你的xcode项目目录中使用`pod spec create RestUI` 来创建一个属于你这个组件的描述文件podspec(cocoapods里面的叫法）。命令运行完已经有sample的了， 具体请参考[这里](http://docs.cocoapods.org/specification.html) 由于配置比较多，这里不多说了。
  * => 配置完podspec文件，记得使用`pod spec lint` 检查下格式是否有错。
  * => 提交个版本之后创建个tag，比如叫0.1（记得你的podspec文件里面的version相关的配置也要叫0.1）
  * => 别人的Podfile只要指定你的仓库地址就能使用了（如果要指定版本号就能使用，必须发一个pull request到CocoaPods项目，等它合并了才能这样）。例如Podfile的内容：

    platform :ios, '5.0' 

    pod 'RestUI', :git => 'https://github.com/jianpx/RestUI.git'
    
    我就是这样做的，开源了一个小组件[AutoCompleteSuffixView](https://github.com/jianpx/AutoCompleteSuffixView), 可以参考下我的podspec文件。

* 如果要使用的第三方包已经有最新的1.2.1版本了，但是官方CocoaPods还没来得及合并，目前最新仍然是1.2.0，怎么破？
  * => 只要在Podfile里面这样指定即可：（找出对应1.2.1版本的commit id）
`pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :commit => '082f8319af'`

* 如果想要用CocoaPods管理本地闭源（暂时不能开源的）包怎么破？
  * => 只需要将这个包放到一个git仓库（甚至是一个zip包），然后定义好一个属于它的podspec文件，最后别人引用的时候只需要在他的Podfile上面这样写：
`pod 'AFNetworking', :path => '~/mygithub/AFNetworking'`

* 使用CocoaPods要怎么进行版本管理？
  * => 官方推荐是把Podfile 和 Podfile.lock 都纳入版本管理。

* 有没有Xcode的插件集成CocoaPods的大部分功能的？
  * => 答案是有的，如果你的Xcode是Xcode5，那么请试用这个：[KFCocoaPodsPlugin](https://github.com/ricobeck/KFCocoaPodsPlugin), 效果如图：![effectiveofkfcocoapodsplugin](https://github.com/ricobeck/KFCocoaPodsPlugin/raw/master/Screenshots/Animation-Completion.gif)

总结
====
有了CocoaPods这个利器，以后要集成一些别人的出色的库或者自己开源一个就很方便了！这里漏了讲一些pod的命令，不过搜索下官网的[pod commands](http://docs.cocoapods.org/commands.html)，一般常用的就是pod install & pod repo update master & pod search  而已。

参考资源
=======
* [CocoaPods Docs](http://docs.cocoapods.org/index.html)
* [CocoaPods进阶：本地包管理](http://www.iwangke.me/2013/04/18/advanced-cocoapods/)
* [Digging into CocoaPods](http://www.cocoanetics.com/2013/01/digging-into-cocoapods/)
* [Cocoapods: Creating a Pod Spec](http://theonlylars.com/blog/2013/01/20/cocoapods-creating-a-pod-spec/)
