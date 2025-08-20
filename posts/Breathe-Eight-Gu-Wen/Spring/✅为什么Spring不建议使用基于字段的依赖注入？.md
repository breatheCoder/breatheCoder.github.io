# 典型回答
在我们通过IDEA编写Spring的代码的时候，假如我们编写了如下代码：



```java
@Autowired
private Bean bean;
```

IDEA会给我们一个warning警告：



> Field injection is not recommended
>



翻阅[官方文档](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/spring-framework-reference/core.html#beans-dependencies)我们会发现：

> Since you can mix constructor-based and setter-based DI, it is a good rule of thumb to use constructors for mandatory dependencies and setter methods or configuration methods for optional dependencies. 
>



大意就是**强制依赖使用构造器注入，可选依赖使用setter注入。**那么这是为什么呢？使用字段注入又会导致什么问题呢？

## 单一职责问题
我们都知道，根据SOLID设计原则来讲，一个类的设计应该符合单一职责原则，就是一个类只能做一件功能，当我们使用基于字段注入的时候，随着业务的暴增，字段越来越多，我们是很难发现我们已经默默中违背了单一职责原则的。



但是如果我们使用基于构造器注入的方式，因为构造器注入的写法比较臃肿，所以它就在间接提醒我们，违背了单一职责原则，该做重构了

```java
@Component
class Test {
    @Autowired
    private Bean bean1;

    @Autowired
    private Bean bean2;

    @Autowired
    private Bean bean3;

    @Autowired
    private Bean bean4;
}
```

```java
@Component
class Test {

    private final Bean bean1;

    private final Bean bean2;

    private final Bean bean3;

    private final Bean bean4;
    
	@Autowired
    public Test(Bean bean1, Bean bean2, 
               Bean bean3, Bean bean4) {
        this.bean1 = bean1;
        this.bean2 = bean1;
 		this.bean3 = bean1;
     	this.bean4 = bean1;
    }
}
```

## 可能产生NPE
对于一个bean来说，它的初始化顺序为：



静态变量或静态语句块 -> 实例变量或初始化语句块 -> 构造方法 -> @Autowired



所以，在静态语句块，初始化语句块，构造方法中使用Autowired表明的字段，都会引起NPE问题

```java
@Component
class Test {
    @Autowired
    private Bean bean;

    private final String beanName;

    public Test() {
        // 此时bean尚未被初始化，会抛出NPE
        this.beanName = bean.getName();
    }
}
```



相反，用构造器的DI，就会实例化对象在使用的过程中，字段一定不为空。

## 隐藏依赖
对于一个正常的使用依赖注入的Bean来说，它应该“显式”的通知容器，自己需要哪些Bean，可以通过构造器通知，public的setter方法通知，这些设计都是没问题的。



外部容器不应该感知到Bean内部私有字段(如上例中的private bean)的存在，私有字段对外部应该是不可见的。由于私有字段不可见，所以在设计层面，我们不应该通过字段注入的方式将依赖注入到私有字段中。这样会破坏封装性。



所以，当我们对字段做注入的时候，Spring就需要关心一个本来被我们封装到一个bean中的私有成员变量，这就和他的封装性违背了。因为我们应该通过setter或者构造函数来修改一个字段的值。

## 不利于测试
很明显，使用了Autowired注解，说明这个类依赖了Spring容器，这让我们在进行UT的时候必须要启动一个Spring容器才可以测试这个类，显然太麻烦，这种测试方式非常重，对于大型项目来说，往往启动一个容器就要好几分钟，这样非常耽误时间。



不过，如果使用构造器的依赖注入就不会有这种问题，或者，我们可以使用Resource注解也可以解决上述问题

[[🔜Autowired和Resource的关系？]]

# 知识扩展
## Spring支持哪些注入方式
### 1. 字段注入
```java
@Autowired
private Bean bean;
```

### 2. 构造器注入
```java
@Component
class Test {
    private final Bean bean;

    @Autowired
    public Test(Bean bean) {
        this.bean = bean;
    }
}
```

### 3. setter注入
```java
@Component
class Test {
    private Bean bean;

    @Autowired
    public void setBean(Bean bean) {
        this.bean = bean;
    }
}
```

## 使用构造器注入可能有哪些问题
如果我们两个bean循环依赖的话，构造器注入就会抛出异常：



```java
@Component
public class BeanTwo implements Bean{

    Bean beanOne;

    public BeanTwo(Bean beanOne) {
        this.beanOne = beanOne;
    }
}
@Component
public class BeanOne implements Bean{

    Bean beanTwo;

    public BeanOne(Bean beanTwo) {
        this.beanTwo = beanTwo;
    }
}
```

> Error creating bean with name 'beanOne': Requested bean is currently in creation: Is there an unresolvable circular reference?
>



如果两个类彼此循环引用，那说明代码的设计一定是有问题的。如果临时解决不了，我们可以在某一个构造器中加入@Lazy注解，让一个类延迟初始化即可。



```java
@Component
public class BeanOne implements Bean{

    Bean beanTwo;

    @Lazy
    public BeanOne(Bean beanTwo) {
        this.beanTwo = beanTwo;
    }
}
```

