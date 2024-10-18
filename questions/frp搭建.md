# 阿里云和 windows11安装frp_0.33.0，实现内网穿透
- - -

## 一、内网穿透目的

实现公网上，访问到windows上启动的web服务

- 架构

  ![image-20241015145308848](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015145308.png)

## 二、内网穿透的环境准备

公网服务器、windows11的电脑、frp软件(需要准备两个软件，一个是安装到公网服务器上的，一个是安装到windows上的)

1. [frp下载地址](https://github.com/fatedier/frp/releases?page=5)

2. 下载版本

   - 此版本(老版本 frp_0.33.0)供参考，比较新的版本，配置文件是与老版本不一样的(最新版本 frp_0.60.0 Win11 下启动报错不知道咋回事，算了。。。)

     - 配置文件区别：frp_0.60.0 配置文件是 .toml格式， frp_0.33.0 配置文件是 .ini 格式

   - 公网服务器上的版本，与windows上的版本是需要对应起来的

     > [frp_0.33.0_linux_amd64.tar.gz](https://github.com/fatedier/frp/releases/download/v0.33.0/frp_0.33.0_linux_amd64.tar.gz)
     >
     > [frp_0.33.0_windows_amd64.zip](https://github.com/fatedier/frp/releases/download/v0.33.0/frp_0.33.0_windows_amd64.zip)

     - ![image-20241015110708142](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015110708.png)

## 三、公网服务器(阿里云)的配置安装

1. 下载 & 解压 & 软链接

   > [frp_0.33.0_linux_amd64.tar.gz](https://github.com/fatedier/frp/releases/download/v0.33.0/frp_0.33.0_linux_amd64.tar.gz)

   ```shell
   wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
   
   tar -zxvf frp_0.33.0_linux_amd64.tar.gz
   
   ln -s frp_0.33.0_linux_amd64 frp
   ```

   ![image-20241015120609164](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015120609.png)

2. 修改配置文件 `frps.ini`

   ```ini
   # [common] is integral section
   [common]
   # A literal address or host name for IPv6 must be enclosed
   # in square brackets, as in "[::1]:80", "[ipv6-host]:http" or "[ipv6-host%zone]:80"
   bind_addr = 0.0.0.0
   bind_port = 7000
   
   # 访问公网的web端口
   vhost_http_port = 8080
   
   # frp服务端管理后台配置
   dashboard_addr = 0.0.0.0
   dashboard_port = 7500
   dashboard_user = admin
   dashboard_pwd = admin
   
   # dashboard assets directory(only for debug mode)
   # assets_dir = ./static
   # console or real logFile path like ./frps.log
   log_file = ./frps.log
   # trace, debug, info, warn, error
   log_level = info
   log_max_days = 3
   # disable log colors when log_file is console, default is false
   disable_log_color = false
   
   # AuthenticationMethod specifies what authentication method to use authenticate frpc with frps.
   # If "token" is specified - token will be read into login message.
   # If "oidc" is specified - OIDC (Open ID Connect) token will be issued using OIDC settings. By default, this value is "token".
   authentication_method = token
   # auth token
   token = 12345678
   ```

3. `systemctl` 管理 frp

   > 或者 手动直接启动 `./frps -c frps.ini`

   ```shell
   # 安装服务并执行启动
   #放到指定文件夹,以能够用systemctl去启动frp
   sudo mkdir -p /etc/frp
   sudo cp frps.ini /etc/frp
   sudo cp frps /usr/bin
   sudo cp systemd/frps.service /usr/lib/systemd/system/
   
   # 开机自启动
   sudo systemctl enable frps
   #启动frps状态
   sudo systemctl start frps
   #查看frps状态
   systemctl status frps.service
   ```

4. 阿里云防火墙开放端口

   ![image-20241015112056534](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015112056.png)

5. 检测 frp 服务端安装是否成功

   > 访问`http://公网ip:7500/`，出现下图时，代表服务端搭建成功

   ![image-20241015113628156](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015113628.png)

## 四、windows客户端的配置安装

1. 下载并解压客户端软件包

   > [frp_0.33.0_windows_amd64.zip](https://github.com/fatedier/frp/releases/download/v0.33.0/frp_0.33.0_windows_amd64.zip)

2. 更改配置文件 ` frpc.ini`

   ```ini
   # [common] is integral section
   [common]
   server_addr = 139.224.206.197
   server_port = 7000
   
   # for authentication
   authentication_method = token
   token = 12345678
   
   # frp 客户端管理后台配置
   admin_addr = 127.0.0.1
   admin_port = 7400
   admin_user = admin
   admin_pwd = admin
   # Admin assets directory. By default, these assets are bundled with frpc.
   # assets_dir = ./static
   
   # Resolve your domain names to [server_addr] so you can use http://web01.yourdomain.com to browse web01 and http://web02.yourdomain.com to browse web02
   [web_8160]
   type = http
   local_ip = 127.0.0.1
   local_port = 8160
   # 或者域名
   custom_domains = 139.224.206.197
   ```
   
3. 启动客户端

   > 在软件包所在页面，文件夹 路径上，输入`cmd`，在弹出的黑窗口中执行`frpc.exe` 或者 `frpc.exe -c frpc.ini`

4. 检测 frp 客户端安装是否成功

   > 访问`http://127.0.0.1:7400/`，出现下图时，代表客户端搭建成功

   ![image-20241015113457041](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015113518.png)

## 五、代理测试

访问 `公网ip:8080`映射到 windows上的 `127.0.0.1:8160`

## 六、frp实现多端口穿透

上面步骤实现的是单个端口，那如何实现多端口同时穿透

- 一般有两种方式：
  - 1. 公网IP方式，不同端口分发；
  - 2. 域名方式，根据域名转发；

​			本例使用域名方式举例，实现 `域名:8080`映射到windows上的`127.0.0.1：8160`，同时实现 `域名:8080`映射到windows上的`127.0.0.1：8161`。

1. 只需更改客户端配置文件`frpc.ini`，服务端配置文件不变，其他如上：

   > 注：frp.bkhech.top， frp2.bkhech.top 都解析到 服务端所在的公网IP上

```ini
# [common] is integral section
[common]
server_addr = 139.224.206.197
server_port = 7000

# for authentication
authentication_method = token
token = 12345678

# frp 客户端管理后台配置
admin_addr = 127.0.0.1
admin_port = 7400
admin_user = admin
admin_pwd = admin
# Admin assets directory. By default, these assets are bundled with frpc.
# assets_dir = ./static


# Resolve your domain names to [server_addr] so you can use http://web01.yourdomain.com to browse web01 and http://web02.yourdomain.com to browse web02
[web_8160]
type = http
local_ip = 127.0.0.1
local_port = 8160
custom_domains = frp.bkhech.top

# Resolve your domain names to [server_addr] so you can use http://web01.yourdomain.com to browse web01 and http://web02.yourdomain.com to browse web02
[web_8161]
type = http
local_ip = 127.0.0.1
local_port = 8161
custom_domains = frp2.bkhech.top
```

2. 域名解析

   ![image-20241015152533885](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/10/15/20241015152533.png)
   
3. 重启客户端程序，访问：

   ```shell
   curl http://frp.bkhech.top:8080  ---forward to --->  http://127.0.0.1:8160
   curl http://frp2.bkhech.top:8080  ---forward to --->  http://127.0.0.1:8161
   ```




