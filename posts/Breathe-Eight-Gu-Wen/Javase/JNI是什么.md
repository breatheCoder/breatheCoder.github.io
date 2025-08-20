- [x] JNI是什么  [completion:: 2025-08-12]

在 Java 里，**JNI** 全称是 **Java Native Interface**，中文一般叫 _Java 本地接口_。  
它是 Java 虚拟机（JVM）提供的一套 **调用本地代码（Native Code）** 的机制，让你在 Java 程序里直接调用用 **C、C++、汇编** 等非 Java 语言写的代码，也可以反过来从本地代码调用 Java 方法。

---

## 为什么会有 JNI

Java 虽然跨平台，但也有局限，比如：

- **性能瓶颈**：需要直接操作硬件或执行高性能算法时，纯 Java 可能不够快。
    
- **访问底层 API**：比如操作系统的内核函数、驱动、硬件端口等，Java 自己无法直接做到。
    
- **复用已有 C/C++ 库**：有些功能别人用 C 写好了，不想用 Java 重写一遍。
    

JNI 就是 JVM 和本地代码之间的桥梁。

---

## JNI 的调用流程（正向调用）

以 Java 调用 C 代码为例：

1. **编写 Java 声明**  
    用 `native` 关键字声明方法，但不写方法体：
    
    ```java
    public class HelloJNI {
        static {
            System.loadLibrary("hello"); // 加载本地库 hello.dll / libhello.so
        }
    
        public native void sayHello(); // 声明本地方法
    
        public static void main(String[] args) {
            new HelloJNI().sayHello();
        }
    }
    ```
    
2. **生成 C 头文件**  
    编译 Java 类：
    
    ```bash
    javac HelloJNI.java
    javac -h . HelloJNI.java  # 生成 HelloJNI.h
    ```
    
3. **用 C/C++ 实现方法**
    
    ```c
    #include <jni.h>
    #include <stdio.h>
    #include "HelloJNI.h"
    
    JNIEXPORT void JNICALL Java_HelloJNI_sayHello(JNIEnv *env, jobject obj) {
        printf("Hello from C!\n");
    }
    ```
    
4. **编译生成动态库**  
    Linux 下：
    
    ```bash
    gcc -shared -fpic -I${JAVA_HOME}/include -I${JAVA_HOME}/include/linux HelloJNI.c -o libhello.so
    ```
    
    Windows 下：
    
    ```bash
    cl /I"%JAVA_HOME%\include" /I"%JAVA_HOME%\include\win32" /LD HelloJNI.c /Fehello.dll
    ```
    
5. **运行 Java 程序**
    
    ```bash
    java HelloJNI
    ```
    
    输出：
    
    ```
    Hello from C!
    ```
    

---

## JNI 的反向调用

C/C++ 也可以通过 `JNIEnv*` 调用 Java 方法或访问字段，这样可以把 Java 和本地代码互相嵌套。

---

## 使用 JNI 的代价与风险

- **跨平台性降低**：Java 代码一旦依赖 JNI，就要为不同平台编译不同版本的动态库。
    
- **安全风险**：本地代码不受 JVM 安全沙箱保护，可能造成内存泄漏或直接崩溃 JVM。
    
- **调试难度大**：需要同时调试 Java 和 C/C++，问题定位复杂。
    

---

## 替代方案

- **JNA**（Java Native Access）：比 JNI 更简单，不需要写 C 头文件，直接用 Java 调本地库。
    
- **Panama 项目**（Java 21+）：下一代更安全、更高效的本地调用 API。
    
- **GraalVM**：可以直接编译 Java 和本地代码为单个可执行文件。