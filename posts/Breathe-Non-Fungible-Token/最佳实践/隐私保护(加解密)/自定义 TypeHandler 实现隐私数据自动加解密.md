我们的项目中，有一些隐私数据，比如用户的姓名、身份证号等，这些都是非常敏感的个人隐私数据，如果明文直接存储到数据库中，会导致一旦被脱库，就会造成隐私泄露。所以我们需要对这些隐私数据进行加解密。

## 解决方案

我们系统中使用了Mybatis作为数据库持久层，因此决定使用Mybatis的TypeHandler来解决该问题。

TypeHandler 作用是在MyBatis 执行数据查询或更新操作时，将数据库中的列值转换为Java 对象，并在将Java 对象写入数据库时执行相反的转换。 它提供了一种灵活且可扩展的方式，用于处理数据库类型与Java 类型之间的映射关系。在这个过程中，我们就可以自己写代码，实现某些特殊字段的加解密了。

## 步骤

1. **创建加解密工具类**：编写一个工具类，用于加密和解密数据。
    
2. **实现自定义TypeHandler**：创建一个实现org.apache.ibatis.type.TypeHandler接口的类，在这个类中编写加解密逻辑。
    
3. **注册TypeHandler**：在MyBatis配置文件中注册自定义的TypeHandler，或者在Mapper接口上直接使用。

### 加解密工具类

我们在代码中，定义了一个工具类，用于实现 AES 加解密。

```java
public class AesUtil {  
  
    private static String key = "uTfe6WtWICU/6rk0Gr7qKrAvHaRvQj+HRaHKvSe9UJI=";  
    private static AES aes = SecureUtil.aes(Base64.getDecoder().decode(key));  
  
    public static String encrypt(String content) {  
        //判空修改  
        if (StringUtils.isBlank(content)) {  
            return content;  
        }  
  
        return aes.encryptHex(content);  
    }  
  
    public static String decrypt(String content) {  
        //判空修改  
        if (StringUtils.isBlank(content)) {  
            return content;  
        }  
  
        return aes.decryptStr(content);  
    }  
}
```

这里面的 AES 算法我们没有自己实现，而是直接使用了 Hutool 中提供的。如果想要自己实现，本文结尾也附上了自己实现的方式的代码。

### 实现自定义TypeHandler

我们需要自定义一个TypeHandler，来实现加解密的逻辑，如下面的

AESEncryptTypeHandler

需要实现BaseTypeHandler接口

```java
public class AesEncryptTypeHandler extends BaseTypeHandler<String> {  
    @Override  
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {  
        // 这里使用你的加密方法进行加密  
        ps.setString(i, encrypt(parameter));  
    }  
  
    @Override  
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {  
        String encrypted = rs.getString(columnName);  
        return encrypted == null ? null : decrypt(encrypted);  
    }  
  
    @Override  
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {  
        String encrypted = rs.getString(columnIndex);  
        return encrypted == null ? null : decrypt(encrypted);  
    }  
  
    @Override  
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {  
        String encrypted = cs.getString(columnIndex);  
        return encrypted == null ? null : decrypt(encrypted);  
    }  
  
    /**  
     * 加密方法  
     * @param data  
     * @return  
     */  
    private String encrypt(String data) {  
        // 实现数据加密逻辑  
        return AesUtil.encrypt(data);  
    }  
  
    /**  
     * 解密方法  
     * @param data  
     * @return  
     */  
    private String decrypt(String data) {  
        // 实现数据解密逻辑  
        return AesUtil.decrypt(data);  
    }  
}
```

### 注册TypeHandler

在mybatis plus 中如果没有自定义sql，可以在实体中设置autoResultMap = true

如果有自定义sql的话就无效，需要在_xml定义的result_Map进行配置TypeHandler:

定义好了TypeHandler 之后，我们需要把他注册到我们需要进行加解密的字段中：

在 User 这个类中增加注解，声明AESEncryptTypeHandler的typeHandler：

使用`@TableField(typeHandler = EncryptDecryptTypeHandler.class)`注解是为了在使用MyBatis Plus时指定某个字段的TypeHandler。因为我们的项目中使用的是MyBatis Plus，所以通过这个注解来简化配置。在这种情况下，MyBatis Plus会在该字段进行读写操作时自动调用指定的TypeHandler, 在Mybatis的使用下是无效的(也就是SQL语句是无效的, 只能用resultMap在typeHandler手动指定)

## 自定义加解密算法

GCM需要
- AES密钥
	- 使用256位AES密钥
- 随机IV
	- 初始化向量（IV）的作用是确保即使多次用同一密钥加密相同数据，加密后的数据也是唯一的
	- 一个12字节的数组来存储IV