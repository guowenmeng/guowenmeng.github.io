# 为什么选择 vultr VPS

之前购买了不同商家的云服务器，有阿里云的VPS，也有腾讯云VPS，还有亚马逊的VPS，最后觉得最好用的还是[`vultr`](https://www.vultr.com/) VPS。

相比较而言性价比高，而且 vultr 是国外的，你懂的，没有什么审核限制。

- 可使用**阿里云的`按量付费`VPS作为临时测试节点**

# 购买并连接到 vultr 服务器

自行搜索。。。

# 在 vultr 上搭建shadowsocks

使用远程连接工具 Xshell，连接到 vultr 服务器之后，接下来几个命令让你快速搭建一个属于自己的ss服务器：

1. 搬瓦工安装 wget
   ```
   yum install wget
   ```
2. 搬瓦工执行安装shadowsocks

```
wget –no-check-certificate -O shadowsocks.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
```

3. 搬瓦工获取shadowsocks.sh读取权限

```
chmod +x shadowsocks.sh
```

4. 设置密码和端口号
   当你输入`./shadowsocks.sh 2>&1 | tee shadowsocks.log`后就可以设置密码和端口号了：

![搬瓦工搭建ss](https://wistbean.github.io/images/ss1.png)

5. 选择加密方式
   设置完密码和端口号之后，我们选择加密方式，这里选择 7：

![搬瓦工搭建ss](https://wistbean.github.io/images/ss2.png)

接着按任意键进行安装。

6. 安装完成
   等一会之后，就安装完成了，它会给你显示你需要连接vpn的信息：

![搬瓦工搭建ss](https://wistbean.github.io/images/ss3.png)

可以看到需要连接ss的ip地址，密码，端口，和加密方式。

将这些信息保存起来，那么这时候你就可以使用它们来科学上网啦。

# 使用Shadowsocks

打开 Shadowsocks 客户端，输入ip地址，密码，端口，和加密方式。接着点击确定，右下角会有个小飞机按钮，右键-->启动代理。

![搬瓦工搭建ss](https://wistbean.github.io/images/ss4.png)

这时候就可以科学上网了。

访问以下 Youtube 和 Google 试试看，速度还可以的：
![搬瓦工搭建ss](https://wistbean.github.io/images/ss5.png)
![搬瓦工搭建ss](https://wistbean.github.io/images/ss6.png)

# 使用BBR加速器

让访问速度加速，飞起来！使用 BBR 加速工具。

## 安装 BBR

wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
## 获取读写权限

chmod +x bbr.sh
## 启动BBR安装

./bbr.sh
接着按任意键，开始安装，坐等一会。安装完成一会之后它会提示我们是否重新启动vps，我们输入 y 确定重启服务器。

重新启动之后，输入 `lsmod | grep bbr` 如果看到 tcp_bbr 就说明 BBR 已经启动了。

再访问一下 Youtube，1080p 超高清，很顺畅不卡顿！

![搬瓦工搭建ss](https://wistbean.github.io/images/ss7.png)

# 相关文章

- https://github.com/clown-coding/vpn
- https://github.com/guowenmeng/new-pac
