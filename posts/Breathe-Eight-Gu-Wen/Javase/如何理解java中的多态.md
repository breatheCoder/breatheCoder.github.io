- [x] 如何理解java中的多态  [completion:: 2025-08-12]

## 典型回答

多态的概念比较简单, 就是同一操作作用域不同的对象, 可以有不同的解释, 产生不同的执行效果.

如果按照这个概念来定义的话, 那么多态应该是一种运行期的状态. 为了实现运行期的多态, 或者说是动态绑定, 需要满足三个条件:

	1. 有类继承或者接口实现
	2. 子类要重写父类的方法
	3. 父类的引用指向子类的对象.

简单来一段代码解释一下:

```java
public class parent {
	public void call() {
		System.out.println("im Parent");
	}
}

public class Son extends Parent {
	public void call() {
		System.out.println("im Son");
	}
}

public class Daughter extends Parent {
	public void call() {
		System.out.println("im Daughter");
	}
}

public class Test {
	public static void main(String[] args) {
		Parent p = new Son();
		Parent p1 = new Daugheter();
	}
}
```

这样就实现了多态,同样是parent类的实力, p.call调用的是son类的实现, p1.call调用的是daughter类的实现.

有人说, 你自己定义的时候不久已经知道p是son, p1是daughter, 但是有的时候用到的对象并不都是自己声明的.

比如Spring中Ioc出来的对象, 你在使用的时候就不知道他是谁, 或者说你可以不用关心他是谁. 根据具体情况而定.

```java
public class PayDomainService {
	@Autowired
	private PayServiceFactory payServiceFactory;

	public void pay(PayRequest requestParam) {
		String payChannel = payRequest.getPayChannel();
		payServiceFactory.getPayService(payChannel).pay(payRequest);
	}
}
```

前面说多态是一种运行期的概念, 还有一种说法, 包括维基百科也认为多态分为静态多态和运行时多态.

一般认为java的函数重载是一种静态多态, 因为他需要在编译器决定具体调用哪个方法.

关于这一点, 不同的人有不同的见解.

我认为, 多态应该是一种运行期特性, java中的重写是多态的体现. 不过也有人提出重载是一种静态多态的想法, 这个问题是StackOverflow等网站有很多人讨论, 但并没有什么定论, 我更加倾向于重载不是多态.

## 扩展知识

### 方法的重载和重写

重载就是函数或方法有同样的名称, 但是参数列表不相同的情形, 这样的同名不同参数的函数或者方法之间, 互相称为重载函数或方法.

```java
class BreatheExample {
	public void display(int a) {
		System.out.println("Got Integer data.");
	}

	public void display(String a) {
		System.out.println("Got String data.");
	}
}
```

重写指的是在java的子类于父类中有两个名称, 参数列表都相同的方法的情况. 由于他们具有相同的方法签名, 所以子类中的新方法将覆盖父类中的老方法.

#### 重写和重载的区别

1. 重载是编译器的概念, 重写是运行期的概念
2. 重载遵循所谓"编译器绑定", 即在编译时根据参数变量的类型来判断应该用哪个方法.
3. 重写遵循所谓"运行期绑定", 即在运行时根据引用变量所指向的实际对象的类型来调用方法.