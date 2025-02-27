#  基于 [Netty](https://netty.io/) 集群的聊天服务方案设计

- - -

> 核心问题：
>
> 1. 离线消息如何存储
> 2. Netty 集群如何构建
> 3. [欢迎访问源码](https://gitee.com/bkhech)

## 高性能聊天服务

### Netty 介绍

- Netty 是一个高性能的 异步事件驱动 的网络应用框架，提供了易于使用的服务器/客户端`API`。

- 并发高  ----  `NIO`（非阻塞 IO）
- 传输快  ----  零拷贝

### Netty 架构

![image-20241108180608119](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/08/20241108180608.png)

### 核心概念

####  网络之阻塞和非阻塞、同步和异步

- 阻塞和非阻塞关注的是**等待结果返回期间，调用方线程的状态**
- 同步和异步关注的是**调用方是否主动获取结果**
- [核心概念总结](https://www.processon.com/embed/661a43e01dbaf808dcb566e9?cid=661a43e01dbaf808dcb566ec)

#### Netty 三种线程模型

> 又叫 Reactor 线程模型，注：Reactor 线程可以看做是一个普通线程

- 单线程模型：**所有的 IO** 操作都由**一个 `NIO` 线程**处理的

  ![image-20241111120405267](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/11/20241111120328.png)

- 多线程模型：由**一组 `NIO`** 线程（Reactor线程池）**处理 IO** 操作，多线程指的是多业务线程

  ![image-20241111120405263](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/11/20241111120405.png)

- 主从线程模型：一组线程池接收请求（主），一组线程池处理 IO（从）

  ![image-20241111120421581](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/11/20241111120421.png)

#### Netty 生命周期/初始化器/处理器/异步操作的结果

> Netty 生命周期，初始化器（`initialzer`）、处理器（handler）、异步操作的结果（future）

- 生命周期

  - ChannelHandler 生命周期：

    > handlerAdded：当把ChannelHandler 添加到ChannelPipeline 中时被调用
    > handlerRemoved：当从ChannelPipeline 中移除ChannelHandler 时被调用
    > exceptionCaught：当处理过程中在ChannelPipeline 中有错误产生时被调用

  - Channel 生命周期

    > channelRegistered：当Channel 已经注册到它的EventLoop 并且能够处理I/O 时被调用
    >
    > channelActive：当Channel 处于活动状态时被调用；Channel 已经连接/绑定并且已经就绪
    >
    > channelRead：当从Channel 读取数据时被调用
    >
    > channelReadComplete：当Channel上的一个读操作完成时被调用（没啥用）
    >
    > `userEventTriggered`：当 `ChannelnboundHandler.fireUserEventTriggered()` 方法被调用时被调用

- [Netty总结](https://www.processon.com/embed/660620cae2d2ef4cf415fbe2?cid=660620cae2d2ef4cf415fbe5)

####  聊天会话管理/多端设备

- 聊天会话管理：

  -  `ChannelGroup` 用于记录和管理所有客户端的 channel 组，
     -  那么如何获取全局唯一的 `ChannelGroup`？，使用单例模式

- 多端设备用户会话：

  -  一旦用户建立连接之后，就立即发送一条消息（消息类型为重连或者第一次连接），将用户 id 和连接 Channel 进行绑定

     ```java
         // 用于多端同时接收消息，允许一个账号多个设备同时在线，比如ipad，ios，Mac等设置同时收到消息
         // key: userId  value: 用户的多个channel
         private static final Map<String, List<Channel>> multiSession = new HashMap<>(32);
     
        // 用于记录用户ID和客户端Channel长id的关联关系
     	// key: 客户端长id  value: userId
         private static final Map<String, String> userChannelIdMap = new HashMap<>(32);
     ```

  -  token 的处理，以允许多端登录

     ```java
     /**
          * 设置用户 token
          *  使用 redis 存储 token，以支持分布式会话
          *  设置过期时间，默认为 7 天
          * @param userId
          * @param platform {@link PlatformEnum}
          * @return 返回设置的 token
          */
         public static String setToken(String userId, PlatformEnum platform) {
             String userToken = platform.getType() + StrUtil.DOT + UUID.randomUUID();
     
             // 本方式只能限制用户在一台设备进行登录
     //        redisOperator.set(RedisKeyEnum.REDIS_USER_TOKEN.getKey() + userId, userToken);
     
             // 本方式运行用户在多端多设备进行登录
             redisOperator.set(RedisKeyConstant.REDIS_USER_TOKEN + userToken, userId, 60 * 60 * 24 * 7);
             return userToken;
         }
     ```

  -  同账户多端设备消息同步

     > 需要处理的问题：
     >
     >  	1. 离线消息如何处理？
     >  		离线消息落库即可，同时带来的问题有，未读消息的标记与处理，以及未读消息的个数。
     >  	2. 场景1. 同账户（消息接受者）多端设备消息发送
     >  	3. 场景2. 同账户（消息发送者）除了当前设备外，将消息发送给其他设备

####  心跳机制（保活机制）

-  客户端处于连接状态，但并没有消息的收发，针对这种不太活跃的连接（一段时间内没有读写操作，时间可自定义，比如：1小时），服务器可以去主动断开不活跃的连接，节约服务器的资源。

   实现：`IdleStateHandler` 结合自定义 Handler（`HeartBeatHandler`）

   -  ```java
      // ChannelInitializer.java	
      // 增加心跳支持。针对客户端，如果在60分钟没有向服务端发送读写心跳(ALL)，则主动断开连接，如果是读空闲或者写空间，不做任何处理
      pipeline.addLast(new IdleStateHandler(
              30,
              30,
              60 * 60));
      pipeline.addLast(new HeartBeatHandler());
      ```

   -  ```java
      // HeartBeatHandler.java
      @Slf4j
      public class HeartBeatHandler extends ChannelInboundHandlerAdapter {
      
          @Override
          public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
              // 判断 evt 是否是 IdleStateEvent (用于触发空闲状态的，包括 读空闲、写空闲、读写空闲)
              if (evt instanceof IdleStateEvent event){
                  switch (event.state()) {
                      case READER_IDLE -> {
                          log.info("进入读空闲状态");
                      }
                      case WRITER_IDLE -> {
                          log.info("进入写空闲状态");
                      }
                      default -> {
                          log.info("进入读写空闲状态");
                          log.error("channel关闭前, clients的数量为：{}", ClientChannelGroup.getInstance().size());
                          Channel channel = ctx.channel();
                          // 关闭无用的 Channel，以防资源浪费
                          channel.close();
                          log.error("channel关闭后, clients的数量为：{}", ClientChannelGroup.getInstance().size());
                      }
                  }
              }
          }
      }
      ```


####  消息收发

> 文字/表情/图片/视频/语音

-  [消息存储方案设计](https://www.processon.com/diagraming/6732c287f0252814e707394f)

-  文字/表情/图片/视频  ----  **都转化成基于文本消息的处理**

   - 表情：一个表情对应唯一一个文字占位符。**每个表情定义一个key值（字符名称），发送消息后再转换成相应的value值（`img`标签）**

   - 图片：图片上传到服务器，传输的是图片的地址链接

   - 视频：视频上传和封面截帧，传输的是视频的视频地址链接和封面地址链接

-  语音

   -  语音转文字

      基于百度智能AI语音识别实现 `com.baidu.aip`，本质上就是将语音发送给云厂商，返回文字，然后客户端将文字消息发送给 消息服务 的过程


## 落地离线消息存储方案

> 分布式消息队列技术

### 聊天消息流程设计

> 通过引入消息队列去解耦，异步通信，削峰，保证消息的可靠传递

- 离线消息存储方案设计

![image-20241118181008640](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/18/20241118181008.png)

- 图片、视频、语音收发设计流程图

![image-20241118181344966](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/18/20241118181345.png)

### 数据表设计

```sql
CREATE TABLE `chat_message` (
  `id` varchar(32) NOT NULL,
  `sender_id` varchar(32) NOT NULL COMMENT '发送者的用户id',
  `receiver_id` varchar(32) NOT NULL COMMENT '接受者的用户id',
  `receiver_type` int DEFAULT NULL COMMENT '消息接受者的类型，可以作为扩展字段',
  `msg` varchar(255) NOT NULL COMMENT '聊天内容',
  `msg_type` int NOT NULL COMMENT '消息类型，有文字类、图片类、视频类...等，详见枚举类',
  `chat_time` datetime NOT NULL COMMENT '消息的聊天时间，既是发送者的发送时间、又是接受者的接收时间',
  `show_msg_date_time_flag` int DEFAULT NULL COMMENT '标记存储数据库，用于历史展示。每超过1分钟，则显示聊天时间，前端可以控制时间长短(扩展字段)',
  `video_path` varchar(128) DEFAULT NULL COMMENT '视频地址',
  `video_width` int DEFAULT NULL COMMENT '视频宽度',
  `video_height` int DEFAULT NULL COMMENT '视频高度',
  `video_times` int DEFAULT NULL COMMENT '视频时间',
  `voice_path` varchar(128) DEFAULT NULL COMMENT '语音地址',
  `speak_voice_duration` int DEFAULT NULL COMMENT '语音时长',
  `is_read` tinyint(1) DEFAULT NULL COMMENT '语音消息标记是否已读未读，true: 已读，false: 未读',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='聊天信息存储表';
```

### 消息队列 MQ

[消息队列知识点](https://www.processon.com/mindmap/66004989c71c473fd0fc7b28)

#### 手动实现 `RabbitMQ`  Connection 连接池

- [x] `RabbitMQ` 的 Java 客户端没有自带连接池，所以需要手动实现。[跳转至实现](#RabbitMQConnectionPool)

### 聊天信息相关

> 保存入库、展示、未读总数展示与清除、语音消息已读签收

- 聊天信息入库：使用 `RabbitMQ`（topic 模式）解耦消息生产者（chat-server）和消费者（main-service）

- 大数据量聊天信息入库方式：分库分表 -- `sharding-jdbc`

  - 带来的问题：消息如何保持一致性，分页查询如何解决（展示）？

- 未读总数展示与清除：

  ```java
  // 通过redis进行累加未读消息
  String redisKey = String.join(StrUtil.COLON, RedisKeyConstant.CHAT_MSG_LIST, receiverId);
  redis.incrementHash(redisKey, senderId, 1);
  
  // 获取未读数量，获取时机：在好友列表聊天页
  String key = String.join(StrUtil.COLON, RedisKeyConstant.CHAT_MSG_LIST, myId);
  Map<Object, Object> hgetall = redis.hgetall(key);
  
  //清除未读数量，清除时机：进入聊天对话框，则清除所有未读标志
  String key = String.join(StrUtil.COLON, RedisKeyConstant.CHAT_MSG_LIST, myId);
  redis.setHashValue(key, oppositeId, "0");
  ```

- 语音消息的已读签收：

  - 针对语音消息有一个是否已读字段，最新发送的消息初始状态为 false，接收者点击已读后，调用接口修改状态为 true.

    ```java
    `is_read` tinyint(1) DEFAULT NULL COMMENT '语音消息标记是否已读未读，true: 已读，false: 未读',
    ```

## 构建聊天服务集群

> 聊天服务集群架构

### Netty 单体架构的问题剖析

1. 单体不是高可用服务

2. 那么能不能把Channel存储起来呢？

  		Netty 中的 Channel 一旦初始化就是唯一的，他是和本地机器码绑定相关联的，序列化到其他的媒介中后，那么这个 Channel就无法反序列化回来，失效了。

### Netty 集群服务的多维度解决方案

[解决方案设计图](https://www.processon.com/diagraming/673c60155b61580d25930fc9)

- 方案一：基于 `Nginx`实现Netty 集群。最简单
- 方案二：基于 `Nacos`实现Netty 集群。引入需要gateway这一套微服务架构，没有现成的微服务环境的话，比较重
- 方案三：基于 `Zookeeper`实现Netty 集群。可以不依赖微服务架构，但客户端需要自己实现一个负载均衡逻辑，去获取其中一个服务器地址。**当前采用本方案去实现**。

### 基于 `Zookeeper` 实现Netty 集群

#### 架构设计方案

![image-20241128154241749](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/28/20241128154241.png)

####  `Zookeeper`核心

- [启动或者leader宕机 选举leader流程](https://www.processon.com/diagraming/65ffb624851a570febeb95c9)

- [`ZAB` 协议](https://www.processon.com/diagraming/66001af0188e2649fd9fa4d8)

#### 动态分配 Netty 集群端口

> netty启动的时候，从redis查找有没有端口，如果没有则使用默认的875端口，如果存在端口，则从最大的端口开始累加10

```java
/**
     * 获取端口
     * @return
     */
public static int selectPort() {
    // 默认的875端口
    final int defaultPort = PropertiesUtil.getDefaultPort();
    final String portKey = NETTY_PORT_KEY;

    Map<String, String> map = jedis.hgetAll(portKey);

    List<Integer> portList = map.keySet().stream()
            .map(Integer::parseInt).toList();
    System.out.println("portList = " + portList);
    // netty启动的时候，从redis查找有没有端口，如果没有则使用默认的875端口，如果存在端口，则从最大的端口开始累加10
    int nettyPort = -1;
    if (CollectionUtils.isEmpty(portList)) {
        nettyPort = defaultPort;
        jedis.hset(portKey, nettyPort + "", initOnlineCounts + "");
    } else {
        // 如果存在端口，则从最大的端口开始累加10
        int currPort = portList.stream().max(Integer::compareTo).get() + 10;
        jedis.hset(portKey, currPort + "", initOnlineCounts + "");
        nettyPort = currPort;
    }
    return nettyPort;
}
```

#### Curator 实现 Netty 服务注册

> 使用 zookeeper 的**有序临时节点**注册当前服务到 zookeeper 节点

```java
/**
     * 注册当前服务到 zookeeper 节点
     * @param nodeName 服务命名空间
     * @param ip
     * @param port
     * @throws Exception
     */
    public static void registerNettyServer(String nodeName, String ip, int port) throws Exception {
        CuratorFramework client = CuratorConfig.getClient();
        String path = nodeName;
        Stat stat = client.checkExists().forPath(path);
        if (stat == null) {
            // 为空就创建节点
            client.create().creatingParentsIfNeeded()
                    // 创建持久节点，因为是根节点
                    .withMode(CreateMode.PERSISTENT)
                    .forPath(path);
        } else {
            log.info("stat:{}", stat);
        }
        // 创建对应的临时有序节点，值放在线人数，初始化默认 0
        NettyServerNode nettyServerNode = new NettyServerNode();
        nettyServerNode.setIp(ip);
        nettyServerNode.setPort(port);
        String nodeJson = JsonUtils.objectToJson(nettyServerNode);
        client.create()
                // 创建临时有序节点
                .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                .forPath(path + "/im-", StrUtil.bytes(nodeJson, StandardCharsets.UTF_8));
        log.info("ip为 {}，端口号为 {} 的 chat-server 的服务已经注册到 zookeeper", ip, port);
    }
```

#### 客户端负载均衡实现

> 基于客户端的负载均衡策略（客户端是当前 Controller，服务端是 zookeeper），和 spring cloud 的负载均衡策略是一致的，都是基于户端的负载均衡策略。像 Nginx 是基于服务端的负载均衡策略。

- 最少连接数

  - 在线人数：根据在线人数统计，每当有一个客户单连接成功后（`发送类型为 MsgTypeEnum.CONNECT_INIT(0, "第一次(或重连)初始化连接") 请求`），更新 zookeeper 中在线客户端的数量，供后面查询的算法实现

  ```java
  /**
   * 处理第一次或者短线重连的消息
   *
   * @param dataContentMap
   */
  public static void handleInitMsg(Map<String, Object> dataContentMap) {
       DataContent dataContent = (DataContent) dataContentMap.get("dataContent");
      Channel channel = (Channel) dataContentMap.get("channel");
      // 当 websocket 初次 open 的时候，初始化 Channel，把channel和userId关联起来
      String senderId = dataContent.getChatMsg().getSenderId();
      String currentChannelId = channel.id().asLongText();
      UserChannelSession.putMultiChannels(senderId, channel);
      log.info("用户id是：{}, 对应的channelList的长度是：{}", senderId, UserChannelSession.getMultiChannels(senderId).size());
      UserChannelSession.putChannelIdOfUser(currentChannelId, senderId);
  
      // 客户端传递过来的节点信息
      NettyServerNode minNode = dataContent.getServerNode();
      //初次链接后，对在线人数进行累加
      ZKUtil.incrementOnlineCount(minNode);
  
      // 获得port和IP，在redis中进行设置关系，以便在客户端断线之后减少在线人数
      // 因为在 ChatHandler.handlerRemoved 触发时，只知道 userId 和 Channel之间的关系，不知道 userId 和 ip:port 的关系，所以此时存储关系
      Jedis jedis = JedisPoolUtil.getJedis();
      jedis.set(senderId, JsonUtils.objectToJson(minNode));
  }
  ```

  - 分布式读写锁

    在实现对在线人数进行累加的时候，存在这样的操作：1. 从 `zookeeper` 读取现有数量，2. 在现有数量的基础上进行 + 1，3. 将数量写回 `zookeeper`， 这个过程是多步骤的，当多线程进行操作的时候，存在数据安全问题，所以需要一个分布式锁对共享资源进行保护。使用 Curator 提供的读写锁`InterProcessReadWriteLock` 实现

    ```java
    CuratorFramework client = CuratorConfig.getClient();
    // 创建读写锁（分布式锁：让当前线程排队执行，保护共享资源的安全操作）
    InterProcessReadWriteLock readWriteLock = new InterProcessReadWriteLock(client, "/rw-lock");
    try {
        // 获取写锁
        readWriteLock.writeLock().acquire();
        
        // TODO 对在线人数进行 +1 操作
        
    catch(Exception e) {
        throw new RuntimeException(e);
    } finally {
        // 释放写锁
    	readWriteLock.writeLock().release();
    }
    ```

#### 端口残留、队列残留清理

如果Netty服务停止，会存在 Netty 端口残留，广播消息队列残留的问题。可以**使用 Curator 监听机制监听 Netty 服务的状态**，如果发现服务停止，则清理残留数据。

> 存在的问题：
>
> 1. 监听服务必须高可用，如果 Netty 服务下线的时候，监听服务不在线，会导致无法删除残留数据的问题

- `SpringBoot` 使用Curator 监听机制，监听 `ZK`节点状态，删除 **残留缓存端口**和**残留广播消息队列**

  > ` CuratorCache..listenable().addListener()`

  ```java
  @Bean(value = "curatorClient")
  public CuratorFramework curatorClient() {
      // 定义重试策略
      RetryPolicy retryPolicy = new ExponentialBackoffRetry(sleepMsBetweenRetry, maxRetries);
      // 声明初始化客户端
      CuratorFramework client = CuratorFrameworkFactory.builder()
              .retryPolicy(retryPolicy)
              .connectionTimeoutMs(connectionTimeoutMs)
              .sessionTimeoutMs(sessionTimeoutMs)
              .namespace(namespace)
              .connectString(host)
              .build();
      // 启动 curator 客户端
      client.start();
      log.info("CuratorFramework 客户端已经成功启动！！！！");
      // 注册监听事件
      addWatcher(NODE_NAME, client);
      return client;
  }
  
  /**
   * 注册节点的事件监听
   *
   * @param path   监听的节点
   * @param client 客户端
   */
  public void addWatcher(String path, CuratorFramework client) {
      CuratorCache curatorCache = CuratorCache.build(client, path);
      curatorCache.listenable().addListener((type, oldData, data) -> {
          log.info("节点更新后的数据、状态: {}", data);
          
          final CuratorCacheListener.Type eventType = type;
          switch (eventType) {
              case NODE_CREATED -> log.info("子节点创建事件");
              case NODE_CHANGED -> log.info("子节点更新事件");
              case NODE_DELETED -> {
                  log.info("子节点删除事件");
                  String oldDataJson = new String(oldData.getData());
                  NettyServerNode oldNode = JsonUtils.jsonToPojo(oldDataJson, NettyServerNode.class);
                  log.info("old path: {}, old node: {}", oldData.getPath(), oldNode);
                  if (Objects.nonNull(oldNode)) {
                      String oldPort = oldNode.getPort() + "";
                      String portKey = NETTY_PORT_KEY;
                      
                      // 删除残留缓存端口
                      redisOperator.hdel(portKey, oldPort);
  
                      // 删除mq中残留的节点
                      String queueName = NETTY_QUEUE_NAME_PREFIX + oldPort;
                      rabbitAdmin.deleteQueue(queueName);
                  }
              }
          }
      });
  ```

#### 集群广播消息

- 设计方案

  > 结合 `MQ`，把消息进行广播到各自节点，随后找到相应的 Channel 去发消息，这样就实现了集群发送聊天消息的效果

  ![image-20241128180710501](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/11/28/20241128154118.png)

- 实现

  - 生产者发送消息（发送消息的 `netty` 服务）

    - ```java
      /**
       * MessagePublisher.java
       *  广播消息
       * @param msg 消息
       * @throws Exception
       */
      public static void sendMsgToOtherNettyServer(String msg) throws Exception {
          RabbitMQConnectUtil connectUtils = new RabbitMQConnectUtil();
          String exchange = properties.getProperty("rabbitmq.chat.message.fanout.exchange");
          connectUtils.sendMsg(msg, exchange, "");
      }
      ```
    
      
    
  - 消费者监听消息（消费消息的 `netty` 服务）

    - 在`netty` 服务启动的时候，启动监听

      ```java
      /**
       * RabbitMQConnectUtil.java
       * 监听队列
       * @param exchangeName 交换机名
       * @param queueName 队列名
       * @throws Exception
       */
      public void listen(String exchangeName, String queueName) throws Exception {
          Connection connection = getConnection();
          Channel channel = connection.createChannel();
      
          // 定义交换机，使用 FANOUT 发布订阅模式(广播模式)
          channel.exchangeDeclare(exchangeName,
                  BuiltinExchangeType.FANOUT,
                  true, false, false, null);
          // 定义队列
          channel.queueDeclare(queueName, true, false, false, null);
          // 声明绑定关系，将队列和交换机绑定，并且指定路由键
          channel.queueBind(queueName, exchangeName, "");
          // 创建消费者
          RabbitMQMessageConsumer consumer = new RabbitMQMessageConsumer(channel, exchangeName);
          /*
           * 开启监听消息
           * queue: 监听的队列名
           * autoAck: 是否自动确认，true：告知mq消费者已经消费的确认通知
           * callback: 回调函数，处理监听到的消息
           */
          channel.basicConsume(queueName, true, consumer);
      }
      ```

      ```java
      /**
       * RabbitMessageConsumer
       *  消费者，消费广播的聊天消息，包括多端处理
       *@author guowm
       *@date 2024/11/28
       */
      @Slf4j
      public class RabbitMQMessageConsumer extends DefaultConsumer {
          private final String exchangeName;
      
          /**
           * Constructs a new instance and records its association to the passed-in channel.
           *
           * @param channel the channel to which this consumer is attached
           * @param exchangeName 交换机名称
           */
          public RabbitMQMessageConsumer(Channel channel, String exchangeName) {
              super(channel);
              this.exchangeName = exchangeName;
          }
      
          /**
           * 重写消息配送方法
           * @param consumerTag 消息的标签（标识）
           * @param envelope  信封（一些信息，比如交换机路由等等信息）
           * @param properties 配置信息
           * @param body 收到的消息数据
           * @throws IOException
           */
          @Override
          public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
      
              String msg = new String(body);
              String exchange = envelope.getExchange();
              log.error("监听到来自mq的消息。exchange：{}，message：{}", exchange, msg);
      
              if (exchange.equalsIgnoreCase(exchangeName)) {
                  DataContent dataContent = JsonUtils.jsonToPojo(msg, DataContent.class);
                  assert dataContent != null;
                  String senderId = dataContent.getChatMsg().getSenderId();
                  String receiverId = dataContent.getChatMsg().getReceiverId();
      
                  // 发送聊天信息给接收用户
                  List<io.netty.channel.Channel> receiverChannels =
                          UserChannelSession.getMultiChannels(receiverId);
                  if (!CollectionUtils.isEmpty(receiverChannels)) {
                      UserChannelSession.sendToTarget(receiverChannels, dataContent);
                  }
      
                  // 同步聊天信息给自己其他设备
                  String currentChannelId = dataContent.getExtend();
                  List<io.netty.channel.Channel> senderChannels =
                          UserChannelSession.getMyOtherChannel(senderId, currentChannelId);
                  if (!CollectionUtils.isEmpty(senderChannels)) {
                      UserChannelSession.sendToTarget(senderChannels, dataContent);
                  }
      
              }
          }
      }
      ```

## 百万-千万-亿级架构

> 10台 Netty 服务集群，在百万，千万级别架构下很常见，7 台 `Zookeeper` 集群是标配，达到这个量级，必须需要这么大的服务级别，在稍微中大型公司很常见

![image-20241202183037917](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/12/02/20241202183038.png)

## 容器化部署

> docker compose 方式

### 基础服务部署

- 比如 Redis 服务

  1. redis.conf

     ```properties
     # redis 密码
     requirepass 123456
     
     # key 监听器配置
     # notify-keyspace-events Ex
     
     # 配置持久化文件存储路径
     dir /redis/data
     # 配置rdb
     # 15分钟内有至少1个key被更改则进行快照
     save 900 1
     # 5分钟内有至少10个key被更改则进行快照
     save 300 10
     # 1分钟内有至少10000个key被更改则进行快照
     save 60 10000
     # 开启压缩
     rdbcompression yes
     # rdb文件名 用默认的即可
     dbfilename dump.rdb
     
     # 开启aof
     appendonly yes
     # 文件名
     appendfilename "appendonly.aof"
     # 持久化策略,no:不同步,everysec:每秒一次,always:总是同步,速度比较慢
     # appendfsync always
     appendfsync everysec
     # appendfsync no
     ```

  2. redis docker-compose.yaml

     ```shell
     https://gitee.com/bkhech/bkhech-chat/blob/master/script/docker/docker-compose-base.yml
     ```

### 微服务部署

- `springboot`微服务

  > 包括：网关服务、文件服务、认证服务、内容服务。以网关服务为例

  1. 编写 `DockerFile` 文件

     > 使用 -Djava.security.egd=file:/dev/./urandom 参数的作用：
     >
     > 问题：tomcat部署项目发现卡在Root WebApplicationContext : initialization completed in xxxms，整个过程没有报错，但是启动时间很长。
     >
     > 原因：由于tomcat启动时产生随机数导致jvm阻塞，可能是多次启动tomcat导致熵池被用空造成阻塞
     >
     > 参考：https://blog.csdn.net/qqnbsp/article/details/120839393

     ```dockerfile
     https://gitee.com/bkhech/bkhech-chat/blob/master/chat-api/auth-service/Dockerfile
     ```
     
  2. 编译镜像文件（Image），并上传（Push）至远程仓库（Registry 比如：阿里云镜像仓库，可选）
  
     ```shell
     https://gitee.com/bkhech/bkhech-chat/blob/master/script/docker/api-build.sh
     https://gitee.com/bkhech/bkhech-chat/blob/master/script/docker/gateway-build.sh
     ```
  
  3. 在生产服务器上，从远程仓库拉取（Pull）镜像文件，并启动项目
  
     ```shell
     https://gitee.com/bkhech/bkhech-chat/blob/master/script/docker/docker-compose-api.yml
     ```
  
- Netty 服务

  > 聊天服务
  
  1. 编写 `DockerFile` 文件
  
     > -jar app.jar ${SERVER_PORT}  端口号当做 JVM 环境变量参数传入
  
     ```dockerfile
     https://gitee.com/bkhech/bkhech-chat/blob/master/chat-api/chat-server/Dockerfile
     ```
  
  2. 编译镜像文件（Image），并上传（Push）至远程仓库（Registry 比如：阿里云镜像仓库，可选）
  
     ```shell
     https://gitee.com/bkhech/bkhech-chat/blob/master/script/docker/api-build.sh
     ```
     
  3. 在生产服务器上，从远程仓库拉取（Pull）镜像文件，并启动项目
  
     ```shell
     https://gitee.com/bkhech/bkhech-chat/blob/master/script/docker/docker-compose-api.yml
     ```

### docker-compose 常用命令

```shell
# docker compose commands

# 后台启动配置文件中的所有服务：
docker compose -f docker-compose.yml -p chat up -d
# 后台启动一部分服务
docker compose -f docker-compose.yml -p chat up -d chat-gateway main-service

# 扩容 main-service 服务。
# 注意：如果要使用扩缩容，不要指定容器名称，让其自动生成(services.main-service.container_name)
# 在 Docker Compose 中，container_name 用于指定容器的名称，通常 Docker Compose 会自动为容器生成名称如果不指定，格式为 <项目名>_<服务名>_<序号>，比如 chat_main_service_1
# 其中，项目名通过 -p 指定
docker compose -f docker-compose.yml -p chat up -d main-service --scale main-service=2

# 关闭所有服务
docker compose down
# 关闭指定服务
docker compose -f docker-compose.yml -p chat down chat-gateway

# 外部需提前创建网络，命令：
docker network create --driver bridge --subnet 172.88.0.0/16 --gateway 172.88.0.1  mynet
#   --driver bridge 网络模式为 桥接模式
#   --subnet 172.88.0.0/16 设置子网
#   --gateway 172.88.0.1 设置网关
#   --mynet 自定义的network名
```

### 注意事项

- restart重启策略 - 生产环境配置推荐
  1. **微服务或高可用服务**:
     使用 **`unless-stopped`** 或 **`always`**，确保服务始终在线，但需配合外部监控工具（如 Prometheus、ELK 等）检查运行状态，防止死循环。
  2. **任务型服务**:
     使用 **`on-failure`** 。

## 遇到的问题

1. SpringCloudGateWay 转发 websocket 请求报错，提示get请求报错500： java.lang.ClassCastException: org.apache.catalina.connector.ResponseFacade cannot be cast to reactor.netty.http.server.HttpServerResponse

   > 我出现这种情况主要是引用的 chat-common 模块中引入了Tomcat 相关的包导致，排除 Tomcat 相关包

   ```xml
   <dependency>
       <groupId>com.bkhech</groupId>
       <artifactId>chat-common</artifactId>
       <version>1.0-SNAPSHOT</version>
       <exclusions>
           <exclusion>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </exclusion>
           <exclusion>
               <groupId>javax.servlet</groupId>
               <artifactId>javax.servlet-api</artifactId>
           </exclusion>
           <exclusion>
               <groupId>org.apache.tomcat.embed</groupId>
               <artifactId>tomcat-embed-core</artifactId>
           </exclusion>
           <exclusion>
               <groupId>javax.servlet</groupId>
               <artifactId>jstl</artifactId>
           </exclusion>
           <exclusion>
               <groupId>org.apache.tomcat.embed</groupId>
               <artifactId>tomcat-embed-jasper</artifactId>
           </exclusion>
       </exclusions>
   </dependency>
   ```

   - 查找 Tomcat 依赖包方法

     - 利用idea自带的jar包依赖来查看当前项目引用情况，具体ctrl+shift+alt+u来查看依赖情况，找到上面的jar包，去除依赖，或者如下图：

       ![image-20241218175323496](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2024/12/18/20241218175329.png)

2. 

## 待办事项

- [x] 使用 `Nacos api` 手动注册聊天服务，然后使用 `Openfeign` 替换聊天服务中的 http

- [ ] 各服务之间限流功能

- [ ] 聊天表分库分表功能，同时带来的分布式事务问题

- [x] 池化技术实现原理
  - [x] ` org.apache.commons.pool2.impl.GenericObjectPoolConfig`
  - [x] 手动实现 `RabbitMQ`  Connection 连接池？
  
- [ ] 换成 `RocketMQ`？

- [ ] Curator 实现分布式锁的原理，提供的锁的类别有哪些？

  - 实现原理
    - 

  - 类别
    - 读写锁：`InterProcessReadWriteLock`
    - 可重入锁：`InterProcessMultiLock`

- [ ] 阿里云语音转文字 `api` 使用实践

###  `Nacos api` 手动注册聊天服务

1. 初始化 `Nacos Client`

   ```java
   /**
    * NacosClient
    *@author guowm
    *@date 2024/12/18
    */
   @Slf4j
   public class NacosClient {
   
       private static volatile NamingService namingService;
   
       private static NamingService initNamingService() {
           final Props properties = PropertiesUtil.getInstance();
   
           properties.setProperty(PropertyKeyConst.NAMESPACE, properties.getProperty("nacos.namespace"));
           properties.setProperty(PropertyKeyConst.SERVER_ADDR, properties.getProperty("nacos.serverAddr"));
           properties.setProperty(PropertyKeyConst.USERNAME, properties.getProperty("nacos.username"));
           properties.setProperty(PropertyKeyConst.PASSWORD, properties.getProperty("nacos.password"));
   
           try {
               // 初始化配置中心的 Nacos Java SDK
               namingService = NacosFactory.createNamingService(properties);
           } catch (Exception e) {
               log.error("初始化 NamingService 失败：{},{}", e, e.getMessage());
               throw new RuntimeException(e);
           }
           return namingService;
   
       }
   
       public static NamingService getNamingService() {
           if (namingService == null) {
               synchronized (NacosClient.class) {
                   if (namingService == null) {
                       namingService = initNamingService();
                   }
               }
           }
           return namingService;
       }
   }
   ```

2. 注册聊天服务

   ```java
    // 把 chat-server 注册到 nacos
           NacosClient.getNamingService().registerInstance("chat-server", ZKUtil.getLocalIp(), nettPort);
   ```

3. 使用服务

   ```java
      final Instance instance = NacosClient.getNamingService().selectOneHealthyInstance("main-service");
                   String isBlackUrl = String.format(properties.getProperty("chat.friend.isBlack.url"), instance.getIp(), instance.getPort());
                   String url = StrUtil.format("{}?friendId1st={}&friendId2nd={}", isBlackUrl, chatMsg.getSenderId(), chatMsg.getReceiverId());
   ```

4. 网关中获取动态聊天服务实例

   ![image-20250117120220050](https://raw.githubusercontent.com/guowenmeng/wodkshje/main/wm2025/01/17/20250117120222.png)

### <span id="RabbitMQConnectionPool">池化技术实现原理</span>

> `org.apache.commons.pool2.impl.GenericObjectPoolConfig`

实战手动实现 `RabbitMQ`  Connection 连接池？

- 为了实现RabbitMQ的Connection连接池，通常会使用如C3P0、HikariCP或Apache Commons Pool等库。这里以Apache Commons Pool2为例，展示如何创建一个简单的RabbitMQ连接池。

1. 添加依赖
    首先，在项目的pom.xml文件中添加Apache Commons Pool2和RabbitMQ客户端的依赖：

  ```xml
  <dependencies>
      <dependency>
          <groupId>org.apache.commons</groupId>
          <artifactId>commons-pool2</artifactId>
          <version>2.11.1</version>
      </dependency>
      <dependency>
          <groupId>com.rabbitmq</groupId>
          <artifactId>amqp-client</artifactId>
          <version>5.14.2</version>
      </dependency>
  </dependencies>
  ```

2. 创建连接池工厂
    接下来，创建一个连接池工厂类，用于配置和创建RabbitMQ的连接：

  ```java
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import org.apache.commons.pool2.BasePooledObjectFactory;
  import org.apache.commons.pool2.PooledObject;
  import org.apache.commons.pool2.impl.DefaultPooledObject;
  
  public class RabbitConnectionFactory extends BasePooledObjectFactory<Connection> {
  
      private final ConnectionFactory factory;
  
      public RabbitConnectionFactory() {
          this.factory = new ConnectionFactory();
          this.factory.setHost("localhost");
          this.factory.setPort(5672);
          this.factory.setUsername("guest");
          this.factory.setPassword("guest");
          this.factory.setVirtualHost("/");
      }
  
      @Override
      public Connection create() throws Exception {
          return factory.newConnection();
      }
  
      @Override
      public PooledObject<Connection> wrap(Connection connection) {
          return new DefaultPooledObject<>(connection);
      }
  
      @Override
      public void destroyObject(PooledObject<Connection> p) throws Exception {
          Connection connection = p.getObject();
          if (connection != null && connection.isOpen()) {
              connection.close();
          }
      }
  
      @Override
      public boolean validateObject(PooledObject<Connection> p) {
          Connection connection = p.getObject();
          return connection != null && connection.isOpen();
      }
  }
  ```

3. 配置连接池
    然后，配置连接池的参数并初始化连接池：

  ```java
  import org.apache.commons.pool2.impl.GenericObjectPool;
  import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
  
  public class RabbitConnectionPool {
  
      private final GenericObjectPool<Connection> pool;
  
      public RabbitConnectionPool() {
          GenericObjectPoolConfig<Connection> config = new GenericObjectPoolConfig<>();
          config.setMaxTotal(10); // 最大连接数
          config.setMaxIdle(5);   // 最大空闲连接数
          config.setMinIdle(2);   // 最小空闲连接数
          config.setMaxWaitMillis(10000); // 获取连接的最大等待时间
  
          pool = new GenericObjectPool<>(new RabbitConnectionFactory(), config);
      }
  
      public Connection getConnection() throws Exception {
          return pool.borrowObject();
      }
  
      public void returnConnection(Connection connection) {
          pool.returnObject(connection);
      }
  
      public void close() {
          try {
              pool.close();
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  }
  ```

4. 使用连接池
    最后，可以在应用中使用这个连接池来获取和释放RabbitMQ连接：

  ```java
  public class RabbitMQClient {
  
      private static final RabbitConnectionPool pool = new RabbitConnectionPool();
  
      public static void main(String[] args) {
          try {
              Connection connection = pool.getConnection();
              // 使用连接进行操作
              // ...
  
              // 归还连接
              pool.returnConnection(connection);
          } catch (Exception e) {
              e.printStackTrace();
          } finally {
              // 关闭连接池
              pool.close();
          }
      }
  }
  ```

  
