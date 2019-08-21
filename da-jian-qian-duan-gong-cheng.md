---
description: 基于vue-cli搭建前端工程，并快速上手。
---

# 搭建工程

## 准备工作

环境依赖：[Node.js](https://link.juejin.im/?target=https%3A%2F%2Fnodejs.org%2Fen%2F)；vue官方脚手架： [vue-cli](https://link.juejin.im/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fvue-cli)；

前端工程化得益于Node.js,所以在初始化工程前，需要先安装Node.js。具体怎么安装Node.js和vue-cli的部分就不再具体说明了，查看官方文档按步骤执行即可（安装Node.js会默认安装npm（包管理工具），vue-cli依赖npm来安装）。另外如果对nodejs版本有要求的话，可以使用[nvm](https://github.com/nvm-sh/nvm)来管理多个版本的Node.js。

## vue-cli脚手架初始化工程

针对Vue工程，推荐使用官方推荐的[vue-cli脚手架](https://cli.vuejs.org/zh/)生成，如果对构建不是很了解强烈不建议自己从零搭建工程，但可以在vue-cli脚手架生成的工程基础上修改，根据自身项目特性作出必要的优化。执行**vue-cli webpack vue-demo**命令生成vue-demo工程。

![vue-cli&#x521D;&#x59CB;&#x5316;&#x5DE5;&#x7A0B;&#x7ED3;&#x6784;&#x56FE;](.gitbook/assets/image%20%288%29.png)

## 安装依赖

> Yarn 是一个包管理器， 你可以通过它使用全世界开发者的代码，或者分享自己的代码。官方地址：[https://yarnpkg.com](https://yarnpkg.com)。

Yarn和npm类似，是一个包管理器，可以通过它来获取全世界开发者的代码，或者分享自己的代码。虽然Node.js自带了npm包管理器，但是在项目中多使用yarn来替代npm。相关比较可以参考[yarn vs npm](https://zhuanlan.zhihu.com/p/23493436)。

使用yarn install或npm install安装依赖，会在工程目录中生成一个node\_modules文件夹，里面存放了所有安装的依赖包。

## 理解package.json

### [https://docs.npmjs.com/files/package.json](https://docs.npmjs.com/files/package.json)

## 业务代码结构

在多人合作的 Vue 项目中，或多或少会使用到 vue-router / vuex / axios 等工具库。基于 vue-cli webpack模板生成的目录结构基础上，建立一个`利于多人合作`、`扩展性强`、`高度模块化`的 vue 项目目录结构。

![&#x9879;&#x76EE;&#x76EE;&#x5F55;&#x7ED3;&#x6784;&#x56FE;](.gitbook/assets/image%20%2813%29.png)

相比vue-cli初始化的项目目录新增了api、common、config、pages、router、store等文件夹。

### api文件夹

用来提供网络处理的基本封装方法\(POST/GET/PUT/DELETE等\)。主要是基于第三方（如axios、superment等）并根据后端接口数据结构特性做二次封装，以减少重复的解析和错误处理工作。

### common文件夹

主要放置公共基础组件等公共资源，

### config文件夹

放置相关配置文件，如常constant.config.js等。

### pages\(views\)文件夹

放置页面相关的组件，这些组件一般和路由相对应，如login.vue、home.vue等。

### router文件夹

路由相关文件。

### store\(vuex\)文件夹

存放和vuex相关的文件，如actions.js、mutations、getter.js、state.js以及modules文件夹等。

**当然也可以根据项目自身特性增减目录结构例如：mixins、filters、directives、util等**。

