在我们的项目中，分库分表使用了Sharding JDBC，而分布式事务使用了 Seata



他们都是和数据库交互的，都需要对我们的 DataSource 做代理，所以在集成的过程中就会出现各种各样的不兼容问题。



其他的都好解决，网上也有答案，但是有一个空指针的问题困扰了我很久，网上也没找到答案。



现象是这样的：



```java
2024-07-25 23:57:01,110 ERROR [DubboServerHandler-30.8.98.40:20882-thread-4 org.springframework.jdbc.support.JdbcTransactionManager - Commit exception overridden by rollback exception
java.lang.NullPointerException: Cannot invoke "io.seata.tm.api.GlobalTransaction.commit()" because the return value of "org.apache.shardingsphere.transaction.base.seata.at.SeataTransactionHolder.get()" is null
        at org.apache.shardingsphere.transaction.base.seata.at.SeataATShardingSphereTransactionManager.commit(SeataATShardingSphereTransactionManager.java:120)
        at org.apache.shardingsphere.transaction.ConnectionTransaction.commit(ConnectionTransaction.java:107)
        at org.apache.shardingsphere.driver.jdbc.core.connection.ConnectionManager.commit(ConnectionManager.java:156)
        at org.apache.shardingsphere.driver.jdbc.core.connection.ShardingSphereConnection.commit(ShardingSphereConnection.java:174)
        at org.springframework.jdbc.datasource.DataSourceTransactionManager.doCommit(DataSourceTransactionManager.java:337)
        at org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:794)
        at org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:757)
        at org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:669)
        at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:419)
        at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:184)

```



当我们接入了 Seata+ShardingJDBC 之后，在分库分表的表进行提交的时候，他会报错，提示一个NullPointerException。

他说是SeataTransactionHolder.get方法返回了一个 null，这是啥呢？



这是shardingsphere-transaction-base-seata-at这个包中提供的一个对 Seata 的全局事务管理的一个工具类。

这个类的代码如下：



```java
final class SeataTransactionHolder {
    
    private static final ThreadLocal<GlobalTransaction> CONTEXT = new ThreadLocal<>();
    
    /**
     * Set seata global transaction.
     *
     * @param transaction global transaction context
     */
    static void set(final GlobalTransaction transaction) {
        CONTEXT.set(transaction);
    }
    
    /**
     * Get seata global transaction.
     *
     * @return global transaction
     */
    static GlobalTransaction get() {
        return CONTEXT.get();
    }
    
    /**
     * Clear global transaction.
     */
    static void clear() {
        CONTEXT.remove();
    }
}

```



这里面其实就是一个 ThreadLocal，里面存放的其实是一个 GlobalTransaction，在 Seata 场景中，就是一个 Seata 的全局事务。



那么SeataTransactionHolder.get方法返回了一个 null，就意味着CONTEXT.get()是 null，也就意味着CONTEXT中没东西。



CONTEXT是在哪被放进去东西的呢？是在set方法中，set 方法在哪被调用了呢？调用过程如下：



```java
public final class ShardingSphereConnection extends AbstractConnectionAdapter {

    @Override
    public void setAutoCommit(final boolean autoCommit) throws SQLException {
        this.autoCommit = autoCommit;
        if (connectionManager.getConnectionTransaction().isLocalTransaction()) {
            processLocalTransaction();
        } else {
			//关注这里
            processDistributeTransaction();
        }
    }

    private void processDistributeTransaction() throws SQLException {
        switch (connectionManager.getConnectionTransaction().getDistributedTransactionOperationType(autoCommit)) {
            case BEGIN:
                connectionManager.close();
				//关注这里
                connectionManager.getConnectionTransaction().begin();
                getConnectionContext().getTransactionConnectionContext().setInTransaction(true);
                break;
            case COMMIT:
                connectionManager.getConnectionTransaction().commit();
                break;
            default:
                break;
        }
    }

}
```



```java
public final class ConnectionTransaction {

	public void begin() {
		//关注这里
	    transactionManager.begin();
	}
}
```



```java
public final class SeataATShardingSphereTransactionManager implements ShardingSphereTransactionManager {

	@Override
	public void begin() {
		//关注这里
	    begin(globalTXTimeout);
	}

    @Override
    @SneakyThrows(TransactionException.class)
    public void begin(final int timeout) {
        if (timeout < 0) {
            throw new TransactionException("timeout should more than 0s");
        }
        Preconditions.checkState(enableSeataAT, "sharding seata-at transaction has been disabled.");
        GlobalTransaction globalTransaction = GlobalTransactionContext.getCurrentOrCreate();
        globalTransaction.begin(timeout * 1000);
		//关注这里
        SeataTransactionHolder.set(globalTransaction);
    }
}
```



以上，我通过`//关注这里` 的标签帮你报名了调用过程。



然后通过 debug 我发现，ConnectionTransaction的 begin方法根本就没有被调用过，原因是ShardingSphereConnection的processDistributeTransaction方法的这个 switch 中，`case BEGIN` 一直就没有满足过条件。



![[cc72c6dd-1d84-4c29-bee1-2a4f2abaa76c.png]]



看一下这个玩意是咋回事儿：

这个switch 的内容是

`connectionManager.getConnectionTransaction().getDistributedTransactionOperationType(autoCommit)` ，看一下`getDistributedTransactionOperationType`方法的实现：



![[cbc82891-2a7d-4fb2-a762-f08bee514f7b.png]]



也就是说，只有`!autoCommit && !transactionManager.isInTransaction() 为true 的时候，才能会直行道这个 BEGIN`。

`isInTransaction`的实现，就在我们今天的主角SeataATShardingSphereTransactionManager中：



![[f5c8e0f6-963e-4e2c-b472-0aaa3299f980.png]]



可以看到，这里是从 RootContext 去获取 XID。



简单介绍下 RootContex 和 XID 是啥。



Seata 的事务上下文由 RootContext 来管理的，应用开启一个全局事务后，RootContext 会自动绑定该事务的 XID，事务结束（提交或回滚完成），RootContext 会自动解绑 XID。



也就是说，一旦全局事务开启了，**<font style="color:rgb(28, 30, 33);">RootContext</font>**<font style="color:rgb(28, 30, 33);">的 XID 就已经有值了，那么也就意味着说，</font>`isInTransaction` 方法将永远返回 true！！！！



纳尼？

没错，这就是我们问题排查下来的根本原因，这个方法永远返回 true，那么将永远不会执行 begin，那么SeataTransactionHolder的get结果将永远都是 null！

这难道不是个 bug？这确定不是个 bug 吗？

还真是个 bug！之前就有人在 github 上提出过这个 issue，而且不止一个：



我在尝试解决这个问题的时候，找到了相关内容，因为看到 issue 是 close 的，我以为解决了，因为ShardingJDBC官方自己说自己解决了。。



![[f8db89fa-8799-46c5-9eb4-6d62f4a737fc.png]]



于是我看了他的这个提交。。。。我只想说，你解决啥了？你就告诉大家在 ShardingJDBC 中不要用`@GlobalTransaction` 就叫解决了？你把 seata 升级到2.0.0就叫解决了？你把 issue 关了就叫解决了？？？



既然他们没解决，那我自己想办法解决吧。

其实，主要问题不就是这个`isInTransaction`方法的逻辑有问题吗，那我们把他改了不就行了么。改成如下内容：



就不要判断去人家 Seata 的`RootContext.getXID()` 去判断了呀，人家一直都是有值的，你判断了有啥用，你应该去你自己的`SeataTransactionHolder.get();` 中去判断啊，第一次没有就老老实实的执行 begin，这样第二次在 commit 的时候不就有了么。



于是，代码中定义一个如下类，注意包路径和类名必须要和人家的保持一致。



然后把isInTransaction方法改了，其他代码原封不动的复制过来。

打包编译，重新运行即可解决。

