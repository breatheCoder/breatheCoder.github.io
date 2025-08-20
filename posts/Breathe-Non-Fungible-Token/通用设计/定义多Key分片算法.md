在我们的项目中，我们对订单表基于买家 ID实现了分库分表，然后用基因法，将分表信息编码到订单号中。也就是说，我们的一次请求，只要带有买家 ID 或者订单号，任意一个字段就可以实现精准路由。



如下面的第12行的配置，声明了两个路由字段：buyer_id, order_id

```yaml
shardingsphere:
  rules:
    sharding:
      tables:
        trade_order:
          actual-data-nodes: ds.trade_order_000${0..3}
          keyGenerateStrategy:
            column: id
            keyGeneratorName: snowflake
          table-strategy:
            complex:
              shardingColumns: buyer_id,order_id
              shardingAlgorithmName: trade-order-sharding
        trade_order_stream:
          actual-data-nodes: ds.trade_order_stream_000${0..3}
          keyGenerateStrategy:
            column: id
            keyGeneratorName: snowflake
          table-strategy:
            complex:
              shardingColumns: buyer_id,order_id
              shardingAlgorithmName: trade-order-sharding
      shardingAlgorithms:
#          t-order-inline:
#            type: INLINE
#            props:
#              algorithm-expression: trade_order_0${Math.abs(buyer_id.hashCode()) % 4}
        trade-order-sharding:
          type: CLASS_BASED
          props:
            algorithmClassName: cn.hollis.nft.turbo.datasource.sharding.algorithm.TurboKeyShardingAlgorithm
            strategy: complex
            tableCount: 4
            mainColum: buyer_id
      keyGenerators:
        snowflake:
          type: SNOWFLAKE
      auditors:
        sharding_key_required_auditor:
          type: DML_SHARDING_CONDITIONS
```



然后，路由算法这里，我们自定义了一个

algorithmClassName:cn.hollis.nft.turbo.datasource.sharding.algorithm.TurboKeyShardingAlgorithm

实现从buyer_id和order_id选一个更合适的作决策。

![[29266881-22db-4c3b-86ab-2bb3410c267f.png]]



这里提到的 mainColum 和tableCount，是我们通过配置文件定义的：



```yaml
trade-order-sharding:
  type: CLASS_BASED
  props:
    algorithmClassName: cn.hollis.nft.turbo.datasource.sharding.algorithm.TurboKeyShardingAlgorithm
    strategy: complex
    tableCount: 4
    mainColum: buyer_id
```

