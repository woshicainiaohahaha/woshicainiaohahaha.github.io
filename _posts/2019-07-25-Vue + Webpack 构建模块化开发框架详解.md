Vue + Webpack 构建模块化开发框架详解[（转载）](https://segmentfault.com/a/1190000017389046?utm_source=tag-newest)
====



### 概要

先在文章开头，做一个总结式的说明，这篇文章主要是讲在前端模块化开发模式中如何用`**Webpack**`这样流行的打包器来为当下一个很火热的框架——`vue.js`，构建一个项目框架。本文例子是一个基本的 Demo，想构建更为复杂和高维护性的框架，请参考 [Webpack 官网 Guide 教程](https://webpack.js.org/guides/)

### 前提

要看懂本文的 Demo，需要你掌握或了解以下内容：

*   _前端传统式开发模式_   VS   _前端模块化开发模式_
*   了解`ES5/ES6`
*   webpack、npm、node.js 之间关系，以及基本概念，这里我写了博客，可以参考[前端模块化开发中 webpack、npm、node、nodejs 之间的关系 [小白总结]](https://blog.csdn.net/AngelLover2017/article/details/84801673)
*   npm 基础知识 (npm 中文文档[前十章](https://www.npmjs.cn/))
*   vue.js 基础知识 (vue 官方网站教程中[介绍](https://cn.vuejs.org/v2/guide/index.html)和[深入了解组件](https://cn.vuejs.org/v2/guide/components-registration.html)以及[工具 / 单文件组件](https://cn.vuejs.org/v2/guide/single-file-components.html))
*   webpack 基础知识 (webpack 官方教程的[概念部分](https://webpack.js.org/concepts/)以及[指南前 6 章](https://webpack.js.org/guides/))

当然还有 htmlcssjs 的基础，但是相信读者如果在查阅 vue 或者 webpack 相关内容，说明已经掌握了基本的技能

### Demo 构建过程框架

1.  确保你的环境正确，笔者这里的环境是`node v10.12.0` ，`npm 6.4.1` ，`webpack 4.27.1`，`webpack-cli 3.1.2`
2.  新建项目目录，并 npm 初始化项目
3.  新建`webpack.config.js`配置文件，并写入项目框架配置
4.  按照各项依赖包 (加载器`loader`，插件`plugin`，`dev`工具等)，包括`vue`，`vue-loader`，`vue-template-compiler`
5.  编写逻辑代码和`*.vue`组件，我们这里仅做了一个示范，没有复杂业务逻辑
6.  启用 webpack 官方的 dev 工具`dev-server`，开启一个开发模式的小服务器
7.  启用 webpack 监听模式，自动构建和编译项目

通过以上 7 步，我们就可以构建一个简单高效的`vue模块化开发`框架了

构建过程
====

```
说明一下：笔者的webpack和webpack-cli都是全局安装的,但有时候会出现某些webpack依赖包`not 
found`的问题，这时候可能是因为webpack4中把webpack-cli工具分离开了，导致可能在全局找不
到cli这时候先执行`npm link webpack` 和 `npm link webpack-cli`将他们加入全局环境执行

```

### npm 初始化项目

cmd

```
mkdir webpackVue
cd webpackVue
npm init

```

根据命令行的提示，填写 package name， version ，description ，entry point 等信息

![](https://segmentfault.com/img/bVbk7yG?w=666&h=497)

构建结果如下：

![](https://segmentfault.com/img/bVbk7yR?w=895&h=302)

### 新建 webpack.config.js，并写入配置

现在的项目结构如下：

![](https://segmentfault.com/img/bVbk7y8?w=261&h=128)

我们来看看这个`webpack.config.js`里都有什么？

webpack.config.js

```
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const VueLoaderPlugin = require('vue-loader/lib/plugin');
const webpack = require('webpack');

module.exports = {
    mode : 'development',
    entry : {
        index : './src/index.js'
    },
    output : {
        filename : '[name].bundle.js',
        path : path.resolve(__dirname,'dist'),
        publicPath : './'
    },
    module : {
        rules : [
            {
                test : /\.css$/,
                use : ['style-loader','css-loader']
            },
            {
                test : /\.html$/,
                loader : 'html-loader'
            },
            {
                test : /\.vue$/,
                loader : 'vue-loader'
            }
        ]
    },
    resolve : {
        alias : {
            vue : 'vue/dist/vue.js'
        }
    },
    plugins : [
        new VueLoaderPlugin(),
        new HtmlWebpackPlugin({
            title : 'vue+webpack模块化开发简例',
            filename : 'index.html',
            template : './src/index.html',
        }),
        new webpack.HotModuleReplacementPlugin(),
        new CleanWebpackPlugin(['dist'])
    ],
    devtool : 'inline-source-map',
    devServer : {
        contentBase : './dist',
        watchContentBase : true,
        compress : true,
        port : 8080,
        hot : true,
        open : true,
    }

}

```

**mode :** 有三个值可以设置，`development`/`production`/`none`，一般来说我们常用的就是`development`和`production`，这里我们设为开发模式

**entry :** 译为入口，这里配置的是我们 webpack 从哪里开始分析我们项目中包的依赖，从哪里开始打包我们的文件，入口可以不只一个，多入口的用法请参见官方文档 [entry points](https://webpack.js.org/concepts/entry-points/), 我这里仅以一个入口为例

**output :** 对应与 entry，这是告诉 webpack 把我们打包后的文件放到哪里，这里的

*   filename 是 打包后文件的名字，[name].bundle.js 其中 [name] 表示原文件的名称
*   path 是 指将我们的项目打包到当前文件夹的 dist 文件夹下
*   publicPath 是 对应于打包后模板中引入的文件的一个公共目录，什么意思呢？比如这里的`./`，通过这个配置，则打包之后的 index.html 文件中引入 js 的路径就会统一加上`./`，这个在下面还会提到👇

**module.rules :** 这里是配置加载`loader`的，简单来说，loader 是一个除 js 资源外的资源加载器，比如常用的`css-loader/style-loader/html-loader`等，默认情况下，webpack 是只能认得 js 文件的，所以就需要一些额外的`loader`来加载不是 js 资源的文件，比如我们这里 vue 的单文件组件是`.vue`后缀，所以我们要引入`vue-loader`(这是 vue 官方给的一个 webpack 加载器)，有关`loder`更多内容请参见官方文档 [Loader](https://webpack.js.org/concepts/loaders/)

**resolve.alias :** 这个是为了解决`import Vue from 'vue';`时默认引入的是 vue.common.js，而非 vue.js。这里是个大坑，Vue 最早会打包生成三个文件，一个是 `runtime only` 的文件 `vue.common.js`，一个是 `compiler only` 的文件 `compiler.js`，一个是 `runtime + compiler` 的文件 `vue.js`。但是 vue 在`package.json`的`main`属性中指向的是`dist/vue.common.js`。这就导致了，构建项目之后会出现，如下错误：

> [Vue warn]: You are using the runtime-only build of Vue where the template compiler is not available. Either pre-compile the templates into render functions, or use the compiler-included build.

关于这个问题，这里有一篇追根溯源的博客可以看一看，[Vue wran:You are using the runtime-only build of Vue...](https://www.cnblogs.com/xiangxinhouse/p/8447507.html)

**plugins :** webpack 本身也是构建在插件之上的，插件的功能可以很强大，做到一些加载器做不到的事情

*   VueLoaderPlugin : 让 vue-loader 发挥魔力的插件，这是官方给的说法
*   HtmlWebpackPlugin : 这是一个可以自动生成 html 的插件，可以免去了我们自己在`./dist`中写一个 html, 更多内容看 npm 社区的介绍 [html-webpack-plugin](https://www.npmjs.com/package/html-webpack-plugin)
*   webpack.HotModuleReplacementPlugin : 启用 webpack 的热模块替换的插件，关于热模块替换，看官方网站 [Hot Module Replacement](https://webpack.js.org/concepts/hot-module-replacement/)
*   CleanWebpackPlugin : 这个插件可以指定每次构建清除之前的构建文件，这个可以让我们的`dist`文件夹里看起来更整洁

**devtool :** 此配置控制是否以及如何生成源映射。这里的`inline-source-map`是相对耗费资源的一个选项，一般只在开发模式使用，这个可以将错误定位到文件和行级，可以很方便的 debug，更多内容 [devTool](https://webpack.js.org/configuration/devtool/)

**devServer :** 这个属性是配置 webpack 的一个官方 dev 工具，`webpack-dev-server`，这是一个小型的服务器，可以在在指定端口运行，为了可以快速的搭建一个应用，不用在用`Node API` 写一个服务器了, 更多内容 [devServer](https://webpack.js.org/configuration/dev-server/)

**_内容还真不少，webpack 的配置虽然繁杂但是灵活性也是很高的，如果不想自己手动搭 vue 的项目框架，vue-cli 可以是你的选择。以上内容，阅读后若不明白，可以去官方文档查阅后，或者你也可以选择继续向下，因为看完整篇内容可以构建一个大概的印象_**

### 创建项目结构，安装各项依赖包

根据我们`webpack.config.js`中的配置，我们的项目结构就像这样一样：

![](https://segmentfault.com/img/bVbk7Ks?w=262&h=200)

其中

*   dist 是 webpack 打包后的文件存放目录
*   src 是我们源文件存放目录

现在，让我们来安装你在 webpack 中配置的各项所需的依赖包 / 加载器 / dev 工具等，安装的命令看起来很简单：

cmd or powershell

```
npm install --save clean-webpack-plugin html-webpack-plugin vue vue-loader

npm install --save-dev css-loader html-loader style-loader vue-template-compiler webpack-dev-server

```

我们这里直接，一次性装好了所有的依赖，当然探索的过程要比这个看似简单的两行艰辛的多。这里要注意`vue-template-compiler`是 vue 编译模板`<template></template>`必需的，不要忘记安装

这里`npm install`的参数`--save`和`--save-dev`的区别就是，save 是生产环境依赖的包，而 save-dev 是开发环境依赖的包，之所以这样分开是为了，既满足开发时便捷工具的需求又满足了真实发布时为了提高性能精简依赖包的需求

好了，我们来看看，安装好依赖后，我们的项目发生了那些变化？

![](https://segmentfault.com/img/bVbk7N1?w=247&h=199)

`package-lock.json`是一个清单文件，我们不需要关注它，我们来看看`package.json`发生了什么

package.json

```
{
  "name": "webpack",
  "version": "1.0.0",
  "description": "vue-router",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "lvbingxu",
  "license": "ISC",
  "dependencies": {
    "clean-webpack-plugin": "^1.0.0",
    "html-webpack-plugin": "^3.2.0",
    "vue": "^2.5.21",
    "vue-loader": "^15.4.2"
  },
  "devDependencies": {
    "css-loader": "^2.0.0",
    "html-loader": "^0.5.5",
    "style-loader": "^0.23.1",
    "vue-template-compiler": "^2.5.21",
    "webpack-dev-server": "^3.1.10"
  }
}


```

我们看到，我们的 package.json 文件随着我们的安装会自动添加上相应的依赖配置，而我们上面所说的`save` ，`save-dev`参数就是分别对应这里的`dependencies`， `devDependencies`，如果 npm 不加参数，默认为`save`参数

> tips : package.json 是一个 json 格式的文件，和 js 的 json 对象支持不一样，请勿在 json 文件中出现不符合 json 格式的字符串，比如注释`//` `/**/`，或是在最后一项后加上`,`

### 编写业务逻辑，使用 Vue 单页组件

现在我们的项目结构就像这样：

![](https://segmentfault.com/img/bVbk7LB?w=251&h=301)

我们来看看源文件代码：

index.html

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Test</title>
</head>
<body>
    <div id="app"></div>
</body>
</html>

```

index.js

```
import Vue from 'vue';
import App from './app.vue';
// 
new Vue(App).$mount('#app');


```

app.vue

vue 官方网站，这里的业务代码很简单，就是将一个`<p></p>`挂载到模板对应的`<div>`上去

好了，到这里我们已经基本把框架构建好了，终于到可以构建一下看看了

### webpack 打包项目，并查看打包后 index.html

cmd or powershell

```
webpack --mode=development

```

以开发模式打包项目，因为在`webpack.config.js`中已经配置过`mode`，所以默认就是开发模式。我们来看看，打包后的项目结构：

![](https://segmentfault.com/img/bVbk7OB?w=252&h=310)

我们来用浏览器打开`/dist/index.html`看看 webpack 是否将 vue 打包成功

**注意**

> 这里由于环境配置不同可能会出现错误：
>
> 1. annot find moudle 'webpack/lib/node/NodeTemplantPlugin'解决办法：我把webpack首先进行了全局安装 cnpm install webpack -g ；然后，又把他进行了局部安装： cnpm install webpack –save-dev ；最后执行：重新做了一次 连接 link：npm link webpack —save-dev
> 2. const CleanWebpackPlugin = require('clean-webpack-plugin')
> 3. typeError:CleanWebpackplugin is not a constructor解决办法：修改webpack.config.js文件里面const CleanWebpackPlugin = require('clean-webpack-plugin')为{CleanWebpackPlugin} = require('clean-webpack-plugin')

![](https://segmentfault.com/img/bVbk7OQ?w=1920&h=1030)

看来我们是成功了，到此为止，我们已经从无到有构建了一个 webpack 打包的 vue 项目，但是他还不完整，我们在做个补充！

### 开启一个 dev-server 服务器，并开启 webpack 的监听模式

为了让开发如虎添翼，我们每次手动的在`cmd or powershell`中输入`webpack --mode=development`或是 `webpack --mode=production`显然是很烦人的，我们可以通过在 package.json 中添加脚本来简化命令，如下：

 ```
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --mode=development",
    "buildp": "webpack --mode=production"
  },

 ```

> tips : 注意这里`"buildp"`的最后不能加上`,`，这对应了上一个 tips 的说法

这样我们可以运行`npm run build`或者`npm run buildp`分别代替`webpack --mode=development` 和`webpack --mode=production`

当然，你发现这还是没有多方便，不急，我们可以开启 webpack 的监听模式，如下：

 ```
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --mode=development",
    "buildp": "webpack --mode=production",
    "watch": "webpack --watch"
  },

 ```

当我们运行`npm run watch`后，这个进程会监听你的文件改变，一旦有文件的改动，webpack 会自动帮你打包项目，无需手动输入了

除此之外，我们还可以开启`dev-server`，一个小型的服务器，并开启热替换模块，这样无需手动刷新浏览器就可以实现视图更新了，是不是很厉害！这个 demo 中使用了热模块替换，但是笔者还未成功，有待进一步研究，下一篇博客再见，这里这是开启了 dev-server 服务器，让我们看看如何开启：

webpack.config.js

 ```
    devServer : {
        contentBase : './dist',
        watchContentBase : true,
        compress : true,
        port : 8080,
        hot : true,
        open : true,
    }

 ```

在你的配置中的最后加入`devServer`的配置，我们再在`package.json`中加入一个脚本：

package.json

 ```
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --mode=development",
    "buildp": "webpack --mode=production",
    "watch": "webpack --watch",
    "server": "webpack-dev-server --open"
  },

 ```

让我们执行，`npm run server`，发现`error`，别急，这是因为我们的`CleanWebpackPlugin`将我们的`dist`文件夹清除了，我们在重新`build`一下

![](https://segmentfault.com/img/bVbk7PY?w=1920&h=1030)

可见，我们的 dev-server 正在监听 8080 端口，we success!

后续
==

  到此，我系统的详细的讲述了，如何使用 webpack 构建一个 vue 的项目框架，希望读者从中受益。我们其实可以看到，我们费了这么大的功夫，其实页面只是一个`<p>Hello VUE!!!</p>`，可能你会说是不是有些小题大作了，拿这个 demo 中的例子来说确实是这样，但是这也是模块化开发和传统式开发的区别，对于小的项目或者需求我们确实不用这么小题大做，但是随着页面的增多，业务需求的扩张和不确定性增大，我们不得不考虑代码的维护和复用，这在传统式开发中很难做到足够优秀的，但是模块化可以