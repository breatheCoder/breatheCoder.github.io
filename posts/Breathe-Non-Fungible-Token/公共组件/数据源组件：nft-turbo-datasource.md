数据源组件，其实就是把数据库的对接这一层封装到一起了，即 mysql、myabtis、mybatis-plus、sharding-jdbc、druid 等。

![[Pasted image 20250819155523.png]]

在配置文件这部分，我们定义了两个配置文件，分别是 datasource.yml 和datasource-sharding.yml。

datasource.yml 用于配置单库单表，datasource-sharding.yml用于配置分库分表。

同时，在我们的项目中用到了 mybatis-plus，有一些通用的配置，如插件、字段的自动填充等，都在这里做了通用的配置：

同时，因为我们还支持分库分表，所以一些通用的和分库分表有关的，比如分布式 ID、分表算法等，也都封装在这个组件中。

