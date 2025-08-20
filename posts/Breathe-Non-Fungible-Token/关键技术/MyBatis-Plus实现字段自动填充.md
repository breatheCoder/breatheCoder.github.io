大家做过 CRUD 的都知道，我们经常会需要有些字段要填充默认值，比如创建时间、修改时间、逻辑删除字段等



要实现这个功能，主要就是实现以下MetaObjectHandler接口，基于这个我们可以很简单的实现这个逻辑。



```java
public class DataObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.setFieldValByNameIfNull("gmtCreate", new Date(), metaObject);
        this.setFieldValByNameIfNull("gmtModified", new Date(), metaObject);
        this.setFieldValByName("deleted", 0, metaObject);
        this.setFieldValByName("lockVersion", 0, metaObject);
    }

    /**
     * 当没有值的时候再设置属性，如果有值则不设置。主要是方便单元测试
     * @param fieldName
     * @param fieldVal
     * @param metaObject
     */
    private void setFieldValByNameIfNull(String fieldName, Object fieldVal, MetaObject metaObject) {
        if (metaObject.getValue(fieldName) == null) {
            this.setFieldValByName(fieldName, fieldVal, metaObject);
        }
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.setFieldValByName("gmtModified", new Date(), metaObject);
    }
}
```



通过以上代码，我们实现了在insert的时候，自动初始化gmtCreate，GMT Modified，deleted以及lockVersion。在update的时候，自动修改gmtModified。



有人说，为什么不直接用数据库的默认值，有些可以，比如lockVersion和deleted，但是有些就不太行，比如每次更新的时候就自动帮我更新创建时间，就没有办法用默认值来实现。



然后再把这个DataObjectHandler配置成一个bean就OK了。

```java
@Configuration
public class DatasourceConfiguration {

    @Bean
    public DataObjectHandler myMetaObjectHandler() {
        return new DataObjectHandler();
    }
}
```

