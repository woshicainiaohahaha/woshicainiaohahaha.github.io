---
layout: article
title: iOS自动打包之fastlane
article_header:
type: overlay
theme: dark
background_color: '#123'
background_image: false
---

这篇文章将会简单介绍iOS项目打包工具fastlane的安装和使用。

### 关于fastlane

> Fastlane是一套使用Ruby写的自动化工具集，旨在简化Android和iOS的部署过程，自动化你的工作流。它可以简化一些乏味、单调、重复的工作，像截图、代码签名以及发布App
> [Github](https://github.com/fastlane/fastlane)、[官网](https://fastlane.tools/)、[文档](https://docs.fastlane.tools/)

**原理：**
> Fastlane是用Ruby语言编写的一套自动化工具集和框架，每一个工具实际都对应一个Ruby脚本，用来执行某一个特定的任务，而Fastlane核心框架则允许使用者通过类似配置文件的形式，将不同的工具有机而灵活的结合在一起，从而形成一个个完整的自动化流程。
> 
> 到目前为止，Fastlane的工具集大约包含170多个小工具，基本上涵盖了打包，签名，测试，部署，发布，库管理等等移动开发中涉及到的内容。

### 为什么要使用fastlane

**使用场景：**

假如：当我们正在进行迭代开发测试，我们会在令人头疼的bug面前来提供新版本给测试人员，一般会执行如下流程：

1. 执行Git Pull命令，拉最新的代码到本地
2. Pod Install安装最新的依赖库
3. 在Xcode中将Build Version增加
4. 在Xcode点击Archive编译并打包
5. 选择输出一个iOS Development模式的ipa文件
6. 繁琐的选择codebit、证书、export
7. 打开蒲公英，将文件上传，保存。OK，假如一切顺利的话，到这里一个新版本的APP已经可以供测试使用。

可是，当我们刚修改完一大堆bug，谁能保证上面的每一个环节都不会出错呢？而且，蛋疼的环节会在出错时导致整个APP不可用。
<div align=center>![坑爹](https://www.runoob.com/wp-content/uploads/2019/01/20190115-1.jpeg)

这时，就是fastlane登场的时候了，你需要做的只有两部：

1. 打开xcode，配置好证书和版本号
2. 配置fastlane（只用配置一次），执行命令fastlane test

好了，你可以去忙别的去了，等手机滴的一声。它已经发短信告诉你，新版本已更新。是不是贼酷炫？贼好用？

下面，我们将开始介绍如何安装以及使用；

### fastlane的安装

> 注意：下面的所有命令都是在终端工具里面使用的。

**一、安装xcode命令行工具**

输入：xcode-select --install，回车。

如果工具已经安装，则会提示：xcode-select: error: command line tools are already installed, use "Software Update" to install updates

如果工具没有安装，则会弹出对话框，点击安装。如下：
<div align=center>![xcode-select](https://img-blog.csdn.net/20150722191049931?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**二、安装fastlane**

输入：sudo gem install fastlane -NV或是brew cask install fastlane，回车。我这里使用gem安装的

安装结束执行fastlane --version，确认一下安装是否完整

### fastlane的使用

**一，初始化fastlane**

1. cd到你的项目目录
2. 输入：fastlane init，回车

这里会弹出四个快捷入口，问你想要使用fastlane做什么？一般情况选择4就可以，我们自己来配置。

```
fastlane init

新版本安装的时候出现了下面的分支选择，按要求选择就行

1. 📸  Automate screenshots
2. 👩‍✈️  Automate beta distribution to TestFlight (自动testfilght型配置)
3. 🚀  Automate App Store distribution (自动发布型配置)
4. 🛠  Manual setup - manually setup your project to automate your (需要手动配置内容)
```

输入：4，回车

如果你的工程没有将Scheme分享出来，将会收到如下错误警告：

<div align=center>![enter image description here](https://upload-images.jianshu.io/upload_images/962634-840d8f655a28e0f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

别急，回到你的工程，打开manage Schemes，然后将工程的share打上对勾：

<div align=center>![xcode-select](https://github.com/woshicainiaohahaha/woshicainiaohahaha.github.io/blob/master/WechatIMG88.jpeg)

然后输入：rm -rf fastlane，将刚才未配置好的文件删除，重新进行fastlane init。

后面根据提示，输入对应的选择条件，一直回车就行。

配置过程可能会出现问题，由于fastlane运行与ruby环境，所以出现问题的可能性有很多。不过，一般不会出现大问题，我们都能在网上找到答案。

配置完毕后，我们回到工程文件目录：

<div align=center>![xcode-select](https://github.com/woshicainiaohahaha/woshicainiaohahaha.github.io/blob/master/WechatIMG89.tiff)

我们可以看到上面的文件信息：
Appfile：主要存放app的app_id team_id app_identifier等信息。
Gemfile：主要定义了该项目软件包依赖的相关事项
Fastfile：是工作文件，我们主要的指令信息都需要配置在该文件里面

**二、关于Fastfile文件的配置**

测试包：

```
default_platform(:ios)

platform :ios do
  desc "打包到pgy"
  lane :test do |options|
    gym(
      clean:true, #打包前clean项目
      export_method: "development", #导出方式
      scheme:"Lemon_HLYZ", #scheme
      configuration: "Debug",#环境
      output_directory:"~/desktop",#ipa的存放目录
      output_name:"2.0"#输出ipa的文件名为当前的build号
      )
      #蒲公英的配置 替换为自己的api_key和user_key
      pgyer(api_key: "api_key", user_key: "user_key",update_description: options[:desc])
  end
end
```

Applestore包：

```
default_platform(:ios)

platform :ios do
     lane :appstore_ipa do
    #下载安装provisioning profile
    sigh(
      username: '3*******3@qq.com',
      app_identifier: 'com.*****.FastlaneDemo1',
      force: true,
      provisioning_name: 'com.*****.FastlaneDemo1 AppStore',
      ignore_profiles_with_different_name: true,
    )
    gym(
      scheme: "FastlaneDemo1",
      export_method: "app-store",
      #如果用到了LLVM混淆工具的话可以指定toolchain(下面是https://github.com/HikariObfuscator/Hikari)：
      #toolchain:"com.naville.hikari",
      output_name:"FastlaneDemo1_AppStore.ipa"
    )
  end
end
```

**三，安装fastlane插件**

如果我们想要传到蒲公英或者其他第三方平台，需要安装对应的插件，这里以蒲公英举例：

[蒲公英官方插件教程](https://www.pgyer.com/doc/view/fastlane)

cd 到工程目录
输入：bundle exec fastlane add_plugin fir

然后根据提示，一路回车就好了。

最后，我们需要在上文提到的Fastfile文件中配置自己的蒲公英账号信息：

```
#蒲公英的配置 替换为自己的api_key和user_key
      pgyer(api_key: "api_key", user_key: "user_key",update_description: options[:desc])
```
### 最后

到目前为止fastlane的基本使用已经介绍完毕了，当然它的用法还有很多，感兴趣可以去深入了解，本文只是给入门同学的一篇指导文章。

参考文章：

[自动化打包之fastlane--(9) 常见错误](https://blog.csdn.net/kuangdacaikuang/article/details/80463858)

[Fastlane带来的全自动化部署](https://www.cnblogs.com/fakeCoder/p/7560456.html)

[Fastlane安装和使用和注意事项](https://www.jianshu.com/p/cdf580a4cd67)
