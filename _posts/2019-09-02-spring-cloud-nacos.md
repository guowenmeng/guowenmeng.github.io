---
layout: post
title: 配置中心Nacos集成实践
date: 2019-09-02
Author: bkhech
tags: [springcloud]
comments: true
---

> 下面主要介绍使用Spring Boot集成Nacos的**服务注册发现**和**配置管理**功能以及**集成规范**。

### 集成规范

- 配置文件目录结构

​      ![](https://gitlab.5fun.com/common-tools/public-library/raw/master/nacos-examples/nacos-spring-boot-example/img/resource_structure.png)

- 在application.properties配置中使用`spring.profiles.active`属性，来区分使用不同环境的配置。
- 开发环境和测试环境可不集成Nacos，或者本地自己搭建Nacos Server

### Spring Boot集成Nacos Server

#### 前提条件

首先安装好Nacos Server服务。线上[Nacos Server地址](http://nacos.abcs8.com/nacos)

#### 集成服务注册发现

> Spring Boot 项目中启动 Nacos 的服务注册发现功能，主要分为以下几步

1.添加依赖

   ```xml
   implementation('com.alibaba.boot:nacos-discovery-spring-boot-starter:0.2.3')
   ```

   `注意：使用0.2.3及以上版本`

2.在application.properties中添加配置

   ```properties
   spring.profiles.active=prod
   #应用名
   spring.application.name=example-boot-service
   ```

3.在application_prod.properties中添加配置

   ```properties
   #自动注册服务
   nacos.discovery.auto-register=true
   #Nacos Server地址
   nacos.discovery.server-addr=nacos.abcs8.com:80
   #命名空间id
   nacos.discovery.namespace=38091048-8a70-40d8-92e7-2eb776f73cb6
   ```

4.启动服务。访问[Nacos Server地址](http://nacos.abcs8.com/nacos)，在`服务管理`->`服务列表`菜单下，`wufan-release`选项卡中，可看到当前启动的服务。如下图所示：

![](https://gitlab.5fun.com/common-tools/public-library/raw/master/nacos-examples/nacos-spring-boot-example/img/service_management.png)

#### 集成配置管理

> Spring Boot 项目中启动 Nacos 的配置管理功能，主要分为以下几步

1.添加依赖

   ```xml
   implementation('com.alibaba.boot:nacos-config-spring-boot-starter:0.2.3')
   ```

   `注意：使用0.2.3及以上版本`

2.在application.properties中添加配置

   ```properties
   spring.profiles.active=prod
   #应用名
   spring.application.name=example-boot-service
   ```

3.在application_prod.properties中添加配置

   ```properties
   #命名空间id
   nacos.config.namespace=38091048-8a70-40d8-92e7-2eb776f73cb6
   #Nacos Server地址
   nacos.config.server-addr=nacos.abcs8.com:80

   #开启配置预加载功能
   nacos.config.bootstrap.log.enable=true
   #配置id
   nacos.config.data-id=example-boot-service
   #开启自动刷新
   #(默认关闭，建议关闭。
   #若开启，需配合Nacos提供的@NacosValue @NacosConfigurationProperties注解使用)
   nacos.config.auto-refresh=false
   #配置文件类型
   nacos.config.type=properties
   ```
4.访问[Nacos Server地址](http://nacos.abcs8.com/nacos)，在`配置管理`->`配置列表`菜单下，`wufan-release`选项卡中， 点击`+`，新建配置 `Data ID`为`nacos.config.data-id` （<!--建议与应用名同名-->）， `配置格式`选择properties类型，将当前application.properties中业务配置（<!--如：数据库配置，Redis配置等-->）剪切到 配置内容中，点击发布按钮。如下图所示：

第一步
![](https://gitlab.5fun.com/common-tools/public-library/raw/master/nacos-examples/nacos-spring-boot-example/img/service_register_discover1.png)

第二步
![](https://gitlab.5fun.com/common-tools/public-library/raw/master/nacos-examples/nacos-spring-boot-example/img/service_register_discover2.png)

5.启动服务。

6.访问业务服务接口，可正常使用，说明配置成功。

**详细请阅** 

- [Nacos官方文档](https://nacos.io/en-us/)