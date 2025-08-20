Seata 是一个分布式事务组件，可以帮我们实现分布式事务。



想要接入他，你需要先按照以下文档部署好中间件：

[[Seata部署]]

然后按照以下步骤进行：

## 项目依赖
```xml
 <dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-all</artifactId>
    <version>2.0.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.antlr</groupId>
            <artifactId>antlr4-runtime</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```



以上，把seata-all和spring-cloud-starter-alibaba-seata给依赖导入到项目中即可，如果项目同时集成了Seata和ShardingJDBC，则需要额外引入以下jar包：



```xml
      <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-transaction-base-seata-at</artifactId>
            <version>5.2.1</version>
        </dependency>
```



但是我们为了解决Seata和ShardingJDBC的兼容问题，额外引入了一个shardingsphere-transaction-core的5.2.1版本，这个具体在下面展开介绍：



[[重写ShardingSphere源码，解决集成Seata的事务失效问题]]


## 初始化表结构
这个 SQL 要在你的应用的数据库（_nft_turbo_）中创建，而不是在 seata 的数据库创建。

```sql
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8mb4 COMMENT ='AT transaction mode undo table';
ALTER TABLE `undo_log` ADD INDEX `ix_log_created` (`log_created`);

```



## 配置文件
在加好了jar包依赖后，就是seata的配置了。在你的application.yml中增加以下内容：



```yaml
seata:
  application-id: ${spring.application.name}
  tx-service-group: default_tx_group # 事务的服务的group，默认用default_tx_group即可
#  use-jdk-proxy: true
#  enable-auto-data-source-proxy: false
  config:
    type: nacos #使用nacos作为配置中心
    nacos:
      server-addr: 114.xx.xxx.45:8848 #使用nacos作为配置中心
      group: seata #nacos上的group
      data-id: seataServer.properties  #nacos上配置的data-id
      namespace: 7ebdfb9b-cd9d-4a5e-8969-1ada0bb9ba04 #nacos上配置的namespace
  registry:  
    type: nacos  #使用nacos作为注册中心
    nacos:
      application: seata-server
      server-addr: 114.xx.xx.45:8848
      group: seata
      cluster: default
      namespace: 7ebdfb9b-cd9d-4a5e-8969-1ada0bb9ba04
```



这里的use-jdk-proxy和enable-auto-data-source-proxy，只需要在使用了shardingJDBC当做数据源的模块中依赖，其他模块中不需要依赖。



```yaml
use-jdk-proxy: true
enable-auto-data-source-proxy: false
```



如果使用了shardingJDBC，同时需要再application.yml的同级目录增加一个seata.conf文件，内容如下：

```nginx
client {
    application.id = nft-turbo-order
    transaction.service.group = default_tx_group
}
```



主要配置的是application.id和transaction.service.group，这个也是只有shardingjdbc代理过的数据源的情况下才需要的。



## 接入成功检查
按照以上配置完成配置之后，就相当于把seata已经接进来了，启动应用如果没有报错的话，到seata的部署服务器上看一下日志，看看是否有注册成功的日志输出：



接下来就可以进行事务的配置了，以AT事务为例。



## AT事务配置


在AT事务中，我们只需要在事务的发起处，用`@GlobalTransactional` 代替`@Transactional` 即可，如：

```java
@GlobalTransactional(rollbackFor = Exception.class)
public boolean paySuccess(PaySuccessEvent paySuccessEvent) {
	//远程调用订单服务

	//远程调用藏品服务

	//远程调用XX服务

	//调用本地的支付服务
}
```



如果参与者没有用ShardingJDBC的话，就啥都不需要额外做，只需要也像前面几步一样把seata接入就行了。

如果某个事务参与者用了shardingjdbc。需要在自己的服务上增加一个额外注解：



```java
@Transactional(rollbackFor = Exception.class)
@ShardingSphereTransactionType(TransactionType.BASE)
public OrderResponse pay(OrderPayRequest request) {
    return doExecuteWithOutTrans(request, tradeOrder -> tradeOrder.pay(request));
}

```



@ShardingSphereTransactionType(TransactionType.BASE)作用是告诉ShardingJDBC，这个方法要加入到seata事务中。

以上，就是完整的接入过程了

