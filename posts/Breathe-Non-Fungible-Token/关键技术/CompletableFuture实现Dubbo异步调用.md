在我们藏品的confirmSale中，我们需要查询用户信息，这地方有些操作不依赖于用户服务，所以在调用userFacadeService的时候，用了CompletableFuture，实现了一个异步请求。



（以下代码已经在项目中移除，调整掉了，原因不是因为不能用，而是我改成了分布式事务，把上链这块操作单独独立出来了，不需要提前查用户了，对性能增益不大。但是这个文档还是保留，用法值得参考）

```java
@Override
@Facade
public CollectionSaleResponse confirmSale(CollectionSaleRequest request) {
    UserQueryRequest userQueryRequest = new UserQueryRequest();
    userQueryRequest.setUserId(Long.valueOf(request.getUserId()));
    CompletableFuture<UserQueryResponse<UserInfo>> queryUserFuture = CompletableFuture.supplyAsync(() -> userFacadeService.query(userQueryRequest));

    CollectionConfirmSaleResponse collectionSaleResponse = collectionService.confirmSale(request);
    CollectionSaleResponse response = new CollectionSaleResponse();

    if (collectionSaleResponse.getSuccess()) {
        Collection collection = collectionSaleResponse.getCollection();
        HeldCollection heldCollection = collectionSaleResponse.getHeldCollection();

        Thread.ofVirtual().start(() -> {
            ChainProcessRequest chainProcessRequest = new ChainProcessRequest();
            chainProcessRequest.setRecipient(getUserInfo(queryUserFuture).getBlockChainUrl());
            chainProcessRequest.setClassId(collection.getClassId());
            chainProcessRequest.setClassName(collection.getName());
            chainProcessRequest.setSerialNo(heldCollection.getSerialNo());
            chainFacadeService.mint(chainProcessRequest);
        });

        response.setSuccess(true);
        response.setHeldCollectionId(heldCollection.getId());
    } else {
        response.setSuccess(false);
        response.setResponseCode(collectionSaleResponse.getResponseCode());
        response.setResponseMessage(collectionSaleResponse.getResponseMessage());
    }

    return response;
}
```



然后在后面真正需要用的地方再取出想要的值就行了：

```java
private UserInfo getUserInfo(CompletableFuture<UserQueryResponse<UserInfo>> queryUserFuture) {
    UserQueryResponse<UserInfo> userQueryResponse;

    try {
        userQueryResponse = queryUserFuture.get();
        if (!userQueryResponse.getSuccess() || null == userQueryResponse.getData()) {
            throw new CollectionException(COLLECTION_USER_QUERY_FAIL);
        }
        return userQueryResponse.getData();
    } catch (InterruptedException | ExecutionException e) {
        throw new RuntimeException(e);
    }
}
```

