# Github Pages自定义域名

- - -
## 背景
Github Pages 可以使你在互联网上建立自己的网站。但是，如果你想要在自己的域名上发，应该如何做呢？本文将介绍如何使用 Github Pages 自定义域名。

## 前置条件
- 您的 Github Pages 能正常访问
- 拥有自己的域名并备案成功

## 操作步骤
### 配置二级域名

   > 记录值就是你的 Github Pages 地址，记录类型填写 CName，主机记录就是你的二级域名地址

![image-20241017102456074](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/17/20241017102456.png)

### 添加 CNAME 文件

   > 文件内容是`二级域名.自己的域名`
   > ![image-20241017102932015](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/17/20241017102932.png)

![image-20241017102750194](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/17/20241017102750.png)
### 访问配置的 CNAME

   > http://blog.bkhech.top
