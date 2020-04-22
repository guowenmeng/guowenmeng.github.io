---
layout: post
title: maven/gradle 查找包依赖树
date: 2019-09-01
Author: bkhech
tags: [构建工具]
comments: true
toc: true
pinned: false
---

## 构建工具之maven gradle

>  查看依赖Jar的关系，解决jar包冲突问题

### Maven 

1. 查看所有依赖：

​        mvn dependency:tree

2. 查看感兴趣的依赖部分：

​       mvn dependency:tree -Dincludes=\*fastjson* 

3. 查看不兴趣的依赖部分：

   mvn dependency:tree -Dexcludes=\*fastjson* 



**注意： 其中：**

​      **“+-”符号表示该包后面还有其它依赖包，**

​      **“\-”表示该包后面不再依赖其它jar包**

​       **-Dincludes/excludes后面的值为groupId，不要传artifactId**

![img](C:\Users\guowm\AppData\Local\Temp\企业微信截图_15790687329249.png)



> 详情参阅

- [查看maven包依赖关系](https://blog.csdn.net/u010003835/article/details/81633093)

### Gradle

1. 查看特定依赖：

   gradlew dependencyInsight --dependency fastjson

2. 查看所有依赖：

   gradlew dependencies
3. 编译指定模块
   gradlew :模块名:clean :模块名:build -x test
   eg: gradlew :web:clean :web:build -x test


> 详情参阅

- [Gradle 用户指南官方文档中文版](https://doc.yonyoucloud.com/doc/wiki/project/GradleUserGuide-Wiki/index.html)
