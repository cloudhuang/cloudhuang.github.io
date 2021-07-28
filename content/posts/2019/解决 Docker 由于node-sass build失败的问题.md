---
title: 解决 Docker 由于node-sass build失败的问题
date: 2019-05-24
categories: ["Node"]
tags: ["Node", "Docker"]
---

### 问题描述：

在Mac上通过docker build node.js的镜像失败，由于`node-sass`安装失败，异常log如下

```
Module build failed (from ./node_modules/_sass-loader@7.1.0@sass-loader/lib/loader.js):
Error: Missing binding /app/node_modules/_node-sass@4.12.0@node-sass/vendor/linux_musl-x64-57/binding.node
Node Sass could not find a binding for your current environment: Linux/musl 64-bit with Node.js 8.x

Found bindings for the following environments:
  - OS X 64-bit with Node.js 11.x

This usually happens because your environment has changed since running `npm install`.
Run `npm rebuild node-sass` to download the binding for your current environment.
    at module.exports (/app/node_modules/_node-sass@4.12.0@node-sass/lib/binding.js:15:13)
    at Object.<anonymous> (/app/node_modules/_node-sass@4.12.0@node-sass/lib/index.js:14:35)
    at Module._compile (module.js:653:30)
    at Object.Module._extensions..js (module.js:664:10)
    at Module.load (module.js:566:32)
    at tryModuleLoad (module.js:506:12)
    at Function.Module._load (module.js:498:3)
    at Module.require (module.js:597:17)
    at require (internal/module.js:11:18)
    at Object.sassLoader (/app/node_modules/_sass-loader@7.1.0@sass-loader/lib/loader.js:46:72)
```

### 原因：

项目的`node_modules`中已经安装了相应的package，Dockerfile中ADD了项目目录到docker中，这样造成了`node-sass`这个包的不一致（系统）

### 解决方法：

网上搜索了很多答案，但是都没有用，其思路基本都是按照提示的信息去rebuild。不过解决起来其实也比较简单：
1. docker build的是时候，先将项目中的`node_moudles`目录删除掉，
2. 或者，在Dockerfile中将`node_modules`删除
3. 重新执行`npm install`


### 一个完整的 Node 的 Dockerfile
```
FROM node:8.16.0-alpine

# install simple http server for serving static content
RUN npm install -g http-server cnpm --registry=https://registry.npm.taobao.org

# make the 'app' folder the current working directory
WORKDIR /app

# copy project files and folders to the current working directory (i.e. 'app' folder)
COPY . .

# install project dependencies
 RUN rm -rf node_modules && cnpm install

# build app for production with minification
RUN cnpm run build-docker

EXPOSE 8080
CMD [ "http-server", "dist" ]
```