基于 `SpringAOP` 抽象分布式锁实现原理

---

### 目的

设计一个类库，以支持不同方案的高性能分布式锁。

预支持方案

- `RedisTemplate`
- `Zookeeper`
- `Redission`

### 思路

1. 抽象分布式组件
   1. 
2. 实现不同方案的 `spring-boot-starter`

### 分布式锁设计

#### 核心`API`

- 注解（@Lock）解析器 -  LockAnnotationParser
- 数据源操作 - LockOperationSource
- 存储分布式锁注解（@Lock）的配置信息 - LockOperation
- 分布式锁模板方法类，提供统一的加锁和释放锁操作 - LockTemplate

#### 集成到Spring 中，以及数据解析

- Spring 分布式锁 @Enable 模块驱动
  - @EnableLock
- 分布式锁之注解
  - @Lock
- 基于注解的解析器
  - SpringLockAnnotationParser，父类是 LockAnnotationParser
  - 解析出  LockOperation 对象
- 基于注解的数据源操作
  - AnnotationLockOperationSource：基于注解操作，父类是  LockOperationSource，依赖 LockAnnotationParser 操作

####  架构关系

```ini
@Lock注解 → LockOperationSource → LockInterceptor → 执行加锁逻辑
```

#### Spring AOP 集成

- 

#### 配置

- 基于Spring Boot 的代理配置 - ProxyLockConfiguration

  
