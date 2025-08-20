### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">1、下载安装包</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">到Seata的官方下载最新的安装包：</font>

<u><font style="color:rgb(27, 154, 238);background-color:rgb(247, 247, 247);">https://seata.apache.org/zh-cn/unversioned/release-history/seata-server</font></u>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">我下载的是2.0.0的版本</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">下载后解压，unzip </font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">unzip seata-server-2.0.0.zip</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);"> ，解压后的目录结构如下：</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

![[f780be9d-deae-4551-ae7b-7d9b21c92ab9.png]]

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">2、配置文件修改</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">进入conf目录，找到application.yml，修改其中的配置：</font>

![[49c51771-f9bb-4498-8fe3-1f2d8076335d.png]]

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">配置中心参考：</font><font style="color:rgb(27, 154, 238);background-color:rgb(247, 247, 247);">https://seata.apache.org/zh-cn/docs/user/configuration/nacos</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">注册中心参考：</font><font style="color:rgb(27, 154, 238);background-color:rgb(247, 247, 247);">https://seata.apache.org/zh-cn/docs/user/registry/nacos</font>

```plain
seata:  config:    # support: nacos, consul, apollo, zk, etcd3    type: nacos    nacos:      server-addr: 139.129.xx.xx:8848      group: 'seata'      namespace: '7ebdfb9b-cd9d-4a5e-8969-1ada0bb9ba04'      #username: 'nacos'      #password: 'nacos'  registry:    # support: nacos, eureka, redis, zk, consul, etcd3, sofa    type: nacos    nacos:      application: seata-server      server-addr: 139.129.xx.xx:8848      group : "seata"      namespace: '7ebdfb9b-cd9d-4a5e-8969-1ada0bb9ba04'      #username: ""      #password: ""      context-path: ""      ##if use MSE Nacos with auth, mutex with username/password attribute      #access-key: ""      #secret-key: ""
```

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">对应的nacos上的namespace配置如下：</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

![[ed16b6b6-d41f-4aa8-91ad-aab0d35db918.png]]

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">注意这里的明明空间ID、和配置文件中的namespace要一致。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">3、在nacos上增加配置信息</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">在nacos上增加配置，namespace选择刚刚创建好的，和seata的配置对应上，创建一个data_id为seataServer.properties的文件</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

![[0ca50810-abfc-4619-8722-e87af8b779f1.png]]

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">文件内容来自：</font><font style="color:rgb(27, 154, 238);background-color:rgb(247, 247, 247);">https://github.com/apache/incubator-seata/blob/develop/script/config-center/config.txt</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">把这里面的内容复制过来，修改其中的部分内容，并配置到seataServer.properties中：</font>

```plain
#For details about configuration items, see https://seata.io/zh-cn/docs/user/configurations.html#Transport configuration, for client and servertransport.type=TCPtransport.server=NIOtransport.heartbeat=truetransport.enableTmClientBatchSendRequest=falsetransport.enableRmClientBatchSendRequest=truetransport.enableTcServerBatchSendResponse=falsetransport.rpcRmRequestTimeout=30000transport.rpcTmRequestTimeout=30000transport.rpcTcRequestTimeout=30000transport.threadFactory.bossThreadPrefix=NettyBosstransport.threadFactory.workerThreadPrefix=NettyServerNIOWorkertransport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandlertransport.threadFactory.shareBossWorker=falsetransport.threadFactory.clientSelectorThreadPrefix=NettyClientSelectortransport.threadFactory.clientSelectorThreadSize=1transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThreadtransport.threadFactory.bossThreadSize=1transport.threadFactory.workerThreadSize=defaulttransport.shutdown.wait=3transport.serialization=seatatransport.compressor=none
#Transaction routing rules configuration, only for the clientservice.vgroupMapping.default_tx_group=default#If you use a registry, you can ignore itservice.default.grouplist=127.0.0.1:8091service.enableDegrade=falseservice.disableGlobalTransaction=false
#Transaction rule configuration, only for the clientclient.rm.asyncCommitBufferLimit=10000client.rm.lock.retryInterval=10client.rm.lock.retryTimes=30client.rm.lock.retryPolicyBranchRollbackOnConflict=trueclient.rm.reportRetryCount=5client.rm.tableMetaCheckEnable=trueclient.rm.tableMetaCheckerInterval=60000client.rm.sqlParserType=druidclient.rm.reportSuccessEnable=falseclient.rm.sagaBranchRegisterEnable=falseclient.rm.sagaJsonParser=fastjsonclient.rm.tccActionInterceptorOrder=-2147482648client.tm.commitRetryCount=5client.tm.rollbackRetryCount=5client.tm.defaultGlobalTransactionTimeout=60000client.tm.degradeCheck=falseclient.tm.degradeCheckAllowTimes=10client.tm.degradeCheckPeriod=2000client.tm.interceptorOrder=-2147482648client.undo.dataValidation=trueclient.undo.logSerialization=jacksonclient.undo.onlyCareUpdateColumns=trueserver.undo.logSaveDays=7server.undo.logDeletePeriod=86400000client.undo.logTable=undo_logclient.undo.compress.enable=trueclient.undo.compress.type=zipclient.undo.compress.threshold=64k#For TCC transaction modetcc.fence.logTableName=tcc_fence_logtcc.fence.cleanPeriod=1h
#Log rule configuration, for client and serverlog.exceptionRate=100
#Transaction storage configuration, only for the server. The file, db, and redis configuration values are optional.store.mode=dbstore.lock.mode=dbstore.session.mode=db#Used for password encryptionstore.publicKey=
#These configurations are required if the `store mode` is `db`. If `store.mode,store.lock.mode,store.session.mode` are not equal to `db`, you can remove the configuration block.store.db.datasource=druidstore.db.dbType=mysqlstore.db.driverClassName=com.mysql.jdbc.Driverstore.db.url=jdbc:mysql://rm-xxxxx:3306/seata?useUnicode=true&rewriteBatchedStatements=truestore.db.user=nfturbostore.db.password=NFTurbo666store.db.minConn=5store.db.maxConn=30store.db.globalTable=global_tablestore.db.branchTable=branch_tablestore.db.distributedLockTable=distributed_lockstore.db.queryLimit=100store.db.lockTable=lock_tablestore.db.maxWait=5000
#Transaction rule configuration, only for the serverserver.recovery.committingRetryPeriod=1000server.recovery.asynCommittingRetryPeriod=1000server.recovery.rollbackingRetryPeriod=1000server.recovery.timeoutRetryPeriod=1000server.maxCommitRetryTimeout=-1server.maxRollbackRetryTimeout=-1server.rollbackRetryTimeoutUnlockEnable=falseserver.distributedLockExpireTime=10000server.xaerNotaRetryTimeout=60000server.session.branchAsyncQueueSize=5000server.session.enableBranchAsyncRemove=falseserver.enableParallelRequestHandle=false
#Metrics configuration, only for the servermetrics.enabled=falsemetrics.registryType=compactmetrics.exporterList=prometheusmetrics.exporterPrometheusPort=9898
```

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">其中需要修改的部分如下，然后保存即可。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

![[edef9ebc-41b6-4e36-9aaa-df628f4c414d.png]]

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">4、创建数据库及表结构</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">在你第三步中指定的那个数据库实例中创建一个库名为seata的数据库，然后执行</font><font style="color:rgb(27, 154, 238);background-color:rgb(247, 247, 247);">https://github.com/apache/incubator-seata/blob/develop/script/server/db/mysql.sql</font><font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);"> 文件中的SQL内容：</font>

```plain
-- -------------------------------- The script used when storeMode is 'db' ---------------------------------- the table to store GlobalSession dataCREATE TABLE IF NOT EXISTS `global_table`(    `xid`                       VARCHAR(128) NOT NULL,    `transaction_id`            BIGINT,    `status`                    TINYINT      NOT NULL,    `application_id`            VARCHAR(32),    `transaction_service_group` VARCHAR(32),    `transaction_name`          VARCHAR(128),    `timeout`                   INT,    `begin_time`                BIGINT,    `application_data`          VARCHAR(2000),    `gmt_create`                DATETIME,    `gmt_modified`              DATETIME,    PRIMARY KEY (`xid`),    KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),    KEY `idx_transaction_id` (`transaction_id`)) ENGINE = InnoDB  DEFAULT CHARSET = utf8mb4;
-- the table to store BranchSession dataCREATE TABLE IF NOT EXISTS `branch_table`(    `branch_id`         BIGINT       NOT NULL,    `xid`               VARCHAR(128) NOT NULL,    `transaction_id`    BIGINT,    `resource_group_id` VARCHAR(32),    `resource_id`       VARCHAR(256),    `branch_type`       VARCHAR(8),    `status`            TINYINT,    `client_id`         VARCHAR(64),    `application_data`  VARCHAR(2000),    `gmt_create`        DATETIME(6),    `gmt_modified`      DATETIME(6),    PRIMARY KEY (`branch_id`),    KEY `idx_xid` (`xid`)) ENGINE = InnoDB  DEFAULT CHARSET = utf8mb4;
-- the table to store lock dataCREATE TABLE IF NOT EXISTS `lock_table`(    `row_key`        VARCHAR(128) NOT NULL,    `xid`            VARCHAR(128),    `transaction_id` BIGINT,    `branch_id`      BIGINT       NOT NULL,    `resource_id`    VARCHAR(256),    `table_name`     VARCHAR(32),    `pk`             VARCHAR(36),    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',    `gmt_create`     DATETIME,    `gmt_modified`   DATETIME,    PRIMARY KEY (`row_key`),    KEY `idx_status` (`status`),    KEY `idx_branch_id` (`branch_id`),    KEY `idx_xid` (`xid`)) ENGINE = InnoDB  DEFAULT CHARSET = utf8mb4;
CREATE TABLE IF NOT EXISTS `distributed_lock`(    `lock_key`       CHAR(20) NOT NULL,    `lock_value`     VARCHAR(20) NOT NULL,    `expire`         BIGINT,    primary key (`lock_key`)) ENGINE = InnoDB  DEFAULT CHARSET = utf8mb4;
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
```

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">5、启动Seata服务端</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">到机器上，先手动创建一个目录：/root/logs/seata/，然后到seata/bin目录下，执行：</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">sh seata-server.sh -h 111.11.11.11</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);"> 命令。通过-h指定本地的ip，</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);"> 111.11.11.11</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">记得换成你自己的ip</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">然后创建一个目录：</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">/root/logs/seata/</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);"> 用于打印日志。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">接下来，打开你的机器的7091和8091端口，通过7091端口可以访问你的seata的管理控制台，输入用户名（默认seata）、密码（默认seata)，即可进入控制台。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">同时在nacos上能看到有一个seata的服务注册上去了：</font>

![[3acc0b29-90ce-4060-b7ed-63b45ed0dd22.png]]

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">常见问题</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">1、执行完启动命令之后，日志正常打印了，但是服务没起来。</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">7091端口无法访问，nacos上也没有对应的服务注册上去。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">a、首先确认下端口是不是开启了，如果没开启，记得开启一下端口。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">b、直接执行一下日志提示中对应的命令，看看报啥错。比如下面这个日志</font>

![]()

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">去掉后面的</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">>> /dev/null 2>&1 &</font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">部分内容，直接执行下这个命令，就能看到具体的报错了。 </font>`<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">/root/package/jdk-21.0.2/bin/java -Dlog.home=/root/logs/seata -server -Dloader.path=/root/package/seata/lib -Xmx2048m -Xms2048m -Xss640k -XX:SurvivorRatio=10 -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=1024m -XX:-OmitStackTraceInFastThrow -XX:-UseAdaptiveSizePolicy -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/root/logs/seata/java_heapdump.hprof -XX:+DisableExplicitGC -Xlog:gc*:file=/root/logs/seata/seata_gc.log:time,tags:filecount=10,filesize=102400 -Dio.netty.leakDetectionLevel=advanced -Dapp.name=seata-server -Dapp.pid=1602373 -Dapp.home=/root/package/seata -Dbasedir=/root/package/seata -Dspring.config.additional-location=/root/package/seata/conf/ -Dspring.config.location=/root/package/seata/conf/application.yml -Dlogging.config=/root/package/seata/conf/logback-spring.xml -jar /root/package/seata/target/seata-server.jar -h 116.62.53.29 -p 8091</font>`

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">我遇到的是</font>

```plain
[0.000s][error][logging] Initialization of output 'file=/root/logs/seata/seata_gc.log' using options 'filecount=10,filesize=102400' failed.
```

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">所以我手动创建了一个/root/logs/seata/目录重启就可以了。</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

### <font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">应用启动报错-service.vgroupMapping.default_tx_group configuration item is required</font>
<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">在你的 seata 的 namespace 中，定义一个配置，名字为：</font><font style="color:rgb(51, 51, 51);background-color:rgba(245, 247, 249, 0.2);">service.vgroupMapping.default_tx_group,</font><font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">值为：default</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

![[7e1825b6-fe1b-473c-acad-7f1b2bfee00c.png]]

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">请注意，group是seata，和代码的配置文件中配置的group要保持一致！！！</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

<font style="color:rgb(38, 38, 38);background-color:rgb(247, 247, 247);">  
</font>

