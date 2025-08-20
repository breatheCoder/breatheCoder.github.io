项目中，经常有各种参数和对象之间的转换，尤其是 DO 转 DTO、转 VO，或者 Request 直接转成Entity 等。



以前的话经常使用hutool工具包的BeanUtil.copProperties，但是这种是基于反射的，这个项目中用到的是性能更高的MapStruct。



需要在pom中添加mapstruct和mapstruct-processor



```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <dependencies>
      
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>1.6.0.Beta1</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>1.18.30</version>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>1.5.5.Final</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>

        </plugins>


</project>

```



加下来定义一个 Convertor：

```java
@Mapper(nullValueCheckStrategy = NullValueCheckStrategy.ALWAYS)
public interface TradeOrderConvertor {

    TradeOrderConvertor INSTANCE = Mappers.getMapper(TradeOrderConvertor.class);

    public TradeOrder mapToEntity(OrderCreateRequest request);

    @Mapping(target = "timeout", expression = "java(request.isTimeout())")
    public TradeOrderVO mapToVo(TradeOrder request);

    public List<TradeOrderVO> mapToVo(List<TradeOrder> request);
}
```

这个Mapper不是mp的mapper，就是mapstruct自带的一个注解，不要写错了。



这里定义了OrderCreateRequest转TradeOrder的方法、TradeOrder转TradeOrderVO的方法。



接下来，使用方式也非常简单：

```java
public static TradeOrder createOrder(OrderCreateRequest request) {
    TradeOrder tradeOrder = TradeOrderConvertor.INSTANCE.mapToEntity(request);
    tradeOrder.setOrderState(TradeOrderState.CREATE);
    tradeOrder.setPaidAmount(BigDecimal.ZERO);
    String orderId = DistributeID.generateWithSnowflake(BusinessCode.TRADE_ORDER, request.getBuyerId());
    tradeOrder.setOrderId(orderId);
    return tradeOrder;
}
```



如上，`TradeOrderConvertor.INSTANCE.mapToEntity(request);`就可以完成参数填充。

