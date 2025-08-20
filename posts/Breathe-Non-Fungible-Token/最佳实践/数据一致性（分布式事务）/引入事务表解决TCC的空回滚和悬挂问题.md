

TCC的方案中，存在着空回滚和悬挂的问题



1. 空回滚问题：TCC的Try过程中，有的参与者成功了，有的参与者失败了，这时候就需要所有参与者全部执行Cancel，这时候，对于那些没有Try成功的参与者来说，本次回滚就是一次空回滚，需要在业务中做好对空回滚的识别和处理。否则就会出现异常报错的问题，甚至一直Cancel失败，最终导致整个分布式事务失败。
2. 悬挂事务问题：TCC的实现方式可能存在悬挂事务问题，在调用TCC事务的一阶段操作时，可能由于网络拥堵而导致超时，此时事务协调器会出发二阶段回滚，调用TCC服务的Cancel操作，在此之后，拥堵在网络上的Try数据包被TCC服务收到，出现了Cancel请求比Try请求先执行的情况。举一个比较常见的例子，一次分布式事务，先发生了Try，但是有的节点失败，又发生了Cancel，而下游的某个节点因为网络延迟先接到了Cancel，在空回滚完成后，又接到了try的请求，然后执行了，这就会导致这个节点的try占用的资源无法释放，也没人会再来处理了，就会导致事务悬挂。



我们是如何解决的呢？我们是创建了一张transaction_log表，然后在cancel和try的时候根据不同情况进行处理。



transaction_log



```sql
CREATE TABLE `transaction_log` (
   `id` bigint NOT NULL AUTO_INCREMENT,
   `gmt_create` datetime NOT NULL COMMENT '创建时间',
   `gmt_modified` datetime NOT NULL COMMENT '更新时间',
   `transaction_id` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '事务id',
   `business_scene` varchar(64) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '业务场景',
   `business_module` varchar(64) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '业务模块',
   `state` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '状态',
   `lock_version` int NULL COMMENT '版本号',
   `deleted` tinyint NULL COMMENT '逻辑删除字段',
   `cancel_type` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL COMMENT 'cancel的类型',
   PRIMARY KEY (`id`),
   KEY `idx_businsess_trans_id`(`transaction_id`,`business_scene`,`business_module`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARACTER SET=utf8mb3 COLLATE=utf8_general_ci COMMENT='事务记录表';
```

以上是transaction_log表的定义，其中：

+ transactionId
+ 事务ID，标记唯一一次事务的，也可以直接用一个唯一的业务id代替，如订单号
+ businessScene
+ 业务场景，具体业务发生的场景，如<下单>就是一个业务场景
+ businessModule
+ 业务模块，一个分布式事务一定是多个模块参与的，所以需要标明具体哪个模块，如订单模块、还是藏品模块，还是盲盒模块
+ state
+ 状态，包含了TRY\CONFIRM\CANCEL三种状态
+ cancelType
+ 取消类型，标明本次如果是CANCEL状态的话，他的取消类型，主要包括
+ CANCEL_AFTER_TRY_SUCCESS：回滚成功-TRY-CANCEL
+ CANCEL_AFTER_CONFIRM_SUCCESS：回滚成功-TRY-CONFIRM-CANCEL
+ EMPTY_CANCEL：空回滚
+ DUPLICATED_CANCEL：幂等成功



## transaction_log和业务表的关系
transaction_log这张表是需要和业务的表在同一个数据库的，因为要借助数据库的本地事务来保证ACID。



假如order模块有自己的数据库，collection也有自己的数据库，那么就要分别到他们的库里面创建这个transaction_log表。



![[19fffe79-5777-4813-a42e-23118381c405.png]]

## 三种情况表中信息
### 如果是一次Try-Cancel的事务（库存 try 成功，订单 try 失败，同时执行cacel），transaction_log内容如下：


ORDER模块：

![[2f132a28-c87c-4532-92de-b49a0c7cf2d0.png]]



（这里之所以是EMPTY_CANCEL而不是CANCEL_AFTER_TRY_SUCCESS，是因为这个case就是藏品模块try成功，但是订单模块try失败，然后进行的CANCEL，如果两个模块都try成功，那就没有cancel啥事儿了。）



COLLECTION模块：

![[9ba97377-1c9c-4030-aa7f-ae14442bd61a.png]]



### **如果是一次Try-Confirm的事务**，transaction_log内容如下：
ORDER模块：

![[ccdc1c41-5dc6-43de-b0c9-4c58d2e99c85.png]]



BLIND_BOX模块

![[ceed39f1-193e-43c9-acd9-ea9b45e8ad81.png]]

### 如果是一次Try-Confirm-Cancel的事务（库存 try 成功，confirm 成功，cancel 成功。订单 try 成功，confirm失败，cancel 成功）
transaction_log内容如下：

ORDER模块

![[7a920071-017c-44fa-9b01-d11c3aae83f5.png]]



这里之所以是CANCEL_AFTER_TRY_SUCCESS而不是CANCEL_AFTER_CONFIRM_SUCCESS，是因为这个case就是盲盒模块confirm成功，但是订单模块confirm失败，然后进行的CANCEL，如果两个模块都CONFIRM成功，那就没有cancel啥事儿了。）



BLIND_BOX模块



![[9d1b254b-2025-4d9b-a173-48cc056bcf01.png]]



## 空回滚代码实现
首先解决空回滚，主要在于可以提前识别并发现空回滚，那么只需要在CANCEL的时候检查下是否有TRY的记录即可。

![[c61fab99-02a2-43f5-90e0-8945a6d50845.png]]



如果发现没有TRY的记录，那么则可以返回一个EMPTY_CANCEL的标记给到调用方，调用方就可以识别这个空回滚不做任何操作。

![[14a77de4-c367-4291-bd23-2be87fa12164.png]]

如上面的订单这里，就是针对EMPTY_CANCEL直接跳过if里面的操作，就实现了一个空回滚

## 悬挂解决代码实现
再仔细看上面的cancelTransaction的代码，你会发现如果是空回滚，我还像数据库中插入了一条CANCEL状态的transaction_log，这个目的就是为了解决悬挂的问题。

![[d941dd1b-7e4f-4703-bd3f-83baf1037c7b.png]]



怎么解决呢，其实悬挂主要是因为TRY的时候无脑try，而我们只要在try的时候检查选，是否发生过cancel，如果发生过，就直接不try了就行了。



![[9287d3f8-93a8-4e8a-a74f-8375507086cc.png]]

