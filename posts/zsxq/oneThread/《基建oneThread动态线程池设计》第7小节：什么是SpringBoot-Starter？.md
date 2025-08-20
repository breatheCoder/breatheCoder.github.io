2025年07月13日 20:07  
什么是SpringBoot-Starter？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：SpringBoot Starter 就像开发者的贴心"料理包"------一行依赖，配料齐全，配置就绪，让你从此告别东拼西凑，专注业务本味。

课程目录如下所示：

*  
引子：SpringBoot 的"小帮手"  
*  
Starter 核心原理揭秘  
*  
为什么会有 Starter？解决了哪些问题？  
*  
SpringBoot 2.x 与 3.x Starter 机制对比  
*  
常用的 SpringBoot 官方 Starters 列表  
*  
最小可运行的自定义 Starter 示例  
*  
  文末总结

引子：SpringBoot 的"小帮手" {#spring-boot}
-----------------------------------

试想一下，在传统开发中我们要使用某个框架功能（比如数据库访问或消息队列），常常需要**东拼西凑** 地添加一堆依赖，还要配置各种参数。这就像下厨做菜前，还得满世界找食材、调料，步骤繁琐又容易出错。而 **Spring Boot Starter** （启动器）就像一个贴心的"大厨助手"或 **预先配好的料理包** ，帮我们一次性备齐所需"食材"（依赖）和默认配置，让开发者**开箱即用，少操很多心** 。Spring Boot Starter 用轻松的话来说，就是 Spring Boot 世界里的"小帮手"，只要把它引入项目，它就会**自动帮你准备好**相关技术栈需要的各种依赖，并且偷偷帮你做好配置工作。结果就是，你只需要专注于业务逻辑，许多底层繁琐的配置都被默默安排妥当了。

举个例子，之前如果想要让项目具备 Web 服务能力，需要引入以下依赖。你开始迷茫：**这么多依赖，我到底应该用哪个？为什么我需要记住这么多东西**？
事实上，在 Spring Boot 出现之前，这种"选择困难症"确实困扰了很多人，包括我。正经人谁会记那么多依赖地址呢。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.x.x.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.x.x.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.x.x.RELEASE</version>
    </dependency>
              
为了解决这个问题，Spring 团队给出了一个非常人性化的方案：**Spring Boot Starter**。

用大白话讲，Starter 就像是一个礼盒。里面装好了你想要的所有东西（依赖和配置），你只需要一个依赖，整个功能全都给你安排妥当。想用 Web 功能吗？加上一个依赖就行了：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
              
发现什么了吗？没错，甚至连版本号都不需要写，因为 SpringBoot 已经帮你统一管理了。

Starter 背景与作用 {#starter}
------------------------

在没有 SpringBoot 的年代，引入第三方组件通常意味着要在 Maven/Gradle 里添加多个坐标依赖，然后再去查文档写大量配置文件，非常麻烦**（依赖版本不匹配、配置分散各处都是常见痛点）** 。SpringBoot 提出了"约定优于配置"的理念，Starters 正是这一理念的产物。**Starter 的作用**简单来说有以下几点：

### 1. 依赖管理一站式 {#1}

一个 Starter 包含了一组相关的依赖，就像"自助套餐"。你加上一个 Starter，相当于引入了一系列经过官方测试搭配好的依赖，避免了自己挨个寻找版本兼容的库。例如，要使用 Spring Data JPA 访问数据库，只需添加 `spring-boot-starter-data-jpa`，它会自动引入 Spring Data JPA、Hibernate 以及数据库连接池等必要组件。不再需要东翻西找依赖，一个 Starter 就**帮你打包好了所有常用库**。

### 2. 自动配置减负 {#2}

Starter **配合 Spring Boot 的自动装配** 机制，能够根据类路径上的依赖自动进行默认配置。这意味着许多以前需要写的样板配置代码，现在 Spring Boot 会替你完成。例如，引入 `spring-boot-starter-web` 后，由于类路径有 Spring MVC 和 Tomcat，应用启动时会自动配置好 **DispatcherServlet、嵌入式 Tomcat 容器等** ，开发者无需再写XML或@Bean去配置这些。Starter **大大减少了配置分散的问题**，将繁琐的配置集中在框架内部约定好，让我们专注于少量必要的属性调整即可。

### 3. 开箱即用的默认 {#3}

大多数 Starter 都提供了一套合理的默认行为。例如 `spring-boot-starter-logging` 会默认使用 Logback 日志框架，`spring-boot-starter-web` 默认使用 Tomcat 容器并开启 Spring MVC。这些默认配置遵循官方**最佳实践** ，避免了开发者从零开始配置。同时如果默认不符合要求，我们仍可以通过 application.properties/yaml 来微调参数，Spring Boot 会自动将配置绑定到对应组件。可以说，Starter **实现了"约定优于配置"**：在有合理默认的前提下，减少开发者的选择成本和配置工作量。

*** ** * ** ***

概括来说，Spring Boot Starter 的出现**解决了依赖管理杂乱和配置碎片化的问题** 。过去可能我们需要拷贝粘贴各种依赖坐标、写很多配置，现在只需引入一个 Starter，**绝大部分配置就帮你默默搞定**。这也是为什么 Spring Boot 能让项目搭建变得如此简单快速的关键原因之一。

Starter 核心原理揭秘 {#starter}
-------------------------

Starters 之所以能做到"引入即用"，背后离不开 **Spring Boot 自动装配（Auto-Configuration）机制** 的支撑。下面让我们以拟人化的方式，跟随 Spring Boot 的启动流程，看看 Starter 是如何把相应功能**自动注入**到应用中的：

![iShot_2025-06-26_20.53.56.png](https://article-images.zsxq.com/Fjfuo6rZGCbc_rKpq65QIYEbWpOH)

当我们在主应用类上标注了 `@SpringBootApplication`（其内部含有`@EnableAutoConfiguration`注解）时，Spring Boot 会在启动时做如下事情：

### 1. 搜集候选自动配置类 {#1}

Spring Boot 利用 `SpringFactoriesLoader` 去扫描应用类路径下所有 JAR 包里的特殊配置文件。对于 Spring Boot 2.x，这个文件就是每个 Starter JAR 中 `META-INF/spring.factories`；而在 Spring Boot 3.x，这个机制有所改变，变成在每个 Starter JAR 中查找 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。这两个文件本质上都是列出该 Starter 提供的**自动配置类列表** 。比如在 Spring Boot 2.x，一个 Starter 可能在 `spring.factories` 中声明：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.MyFeatureAutoConfiguration
              
Spring Boot 读取到这个条目，就知道这个 Starter 包含 `MyFeatureAutoConfiguration` 这个自动配置类需要被加载。而在 Spring Boot 3.x，我们不再使用键值对配置，而是直接在 `AutoConfiguration.imports` 文件里逐行列出自动配置类全限定名，例如：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    com.example.MyFeatureAutoConfiguration
    com.example.AnotherAutoConfiguration
              
Spring Boot 3.x 会扫描 `META-INF/spring/*.imports` 目录下的文件来收集自动配置类名单。这一新机制更简单直观，每行一个类名，不需要再像旧格式那样写键值对，性能和模块化也有所提升。**无论新旧版本**，此步骤的结果都是：根据所有引入的 Starter，Spring Boot 得到了一张"需要尝试自动配置的配置类清单"。

### 2. 按条件装配自动配置类 {#2}

拿到所有候选的自动配置类名后（可能成百上千个），SpringBoot 并不会傻乎乎地把它们统统装配进来，而是会挨个筛选，**判断条件是否满足** 。这得益于 SpringBoot 提供的一系列**条件注解**，常用的如：

*  
`@ConditionalOnClass`：当类路径下存在某个类时才生效。  
*  
`@ConditionalOnMissingClass`：缺少某类时生效。  
*  
`@ConditionalOnBean`：当容器中已有某个 Bean 时/不在时生效。  
*  
  `@ConditionalOnProperty`：当配置文件中某个属性有指定值时生效等等。

这些条件注解被广泛地应用在各个自动配置类上，用于细粒度地控制配置是否需要自动装配。

其中最重要的一个条件是 `@ConditionalOnMissingBean`。它表示"仅当容器中**没有某个特定类型的 Bean** 时，才执行自动配置"。这确保了**当开发者自己定义了同类型的 Bean 时，自动配置会"知趣地"退让**，不会再创建重复的默认Bean。

例如，SpringBoot 会在 JDBC 数据源的自动配置类中用 `@ConditionalOnMissingBean(DataSource.class)` 判断，如果你已经手动提供了一个 `DataSource` Bean，那么默认的自动配置 DataSource 就不会再生效。这种机制保证了 Starter 提供默认配置的同时，不会妨碍你去覆盖/自定义配置。
> **小科普** ：SpringBoot 为了提升启动效率，甚至会在应用启动早期就利用注解处理器预先生成一份关于条件注解的元数据 (`META-INF/spring-autoconfigure-metadata.properties`)。这样在真正逐个评估条件前，可以快速跳过一些明显不满足条件的自动配置，从而加快启动。总之，条件装配机制让自动配置更加智能和高效。

### 3. 注册 Bean 到容器 {#3-bean}

通过条件筛选后，符合条件的自动配置类会被实例化并发挥作用。其实每个自动配置类本质上就是一个加了 `@Configuration` 注解的普通 Spring 配置类，其中定义了一系列 `@Bean` 方法用于注册组件。例如，`WebMvcAutoConfiguration` 会注册 DispatcherServlet、HandlerMapping、HttpMessageConverters 等 Web 开发需要的组件；`DataSourceAutoConfiguration` 会创建数据源连接池 Bean 等等。一旦这些配置类被加载，其内部定义的 Bean 就会按照 Spring 容器的规则被注册。

至此，原本我们需要手工配置的许多 Bean，因为 Starter 的引入而**在后台自动完成注册**了。

*** ** * ** ***

综上所述，SpringBoot Starter 能够自动注入所需功能，**完全是托管于 Spring Boot 自动装配机制** 。Starters 本身通常并不包含太多代码（很多官方 Starter 甚至只是一个 POM 依赖集合），但它们的存在触发了 SpringBoot 去加载对应的 AutoConfiguration **配置模块**。

总之，**Starter 和自动装配是一对好搭档**：Starter 提供依赖和入口，自动装配提供智能配置逻辑，两者结合使得 SpringBoot 应用具有开箱即用的特性。

为什么会有 Starter？解决了哪些问题？ {#starter}
---------------------------------

通过上面的介绍，相信大家已经体会到 Starter 带来的诸多便利。这里我们再**总结提升**一下，Spring Boot Starter 的诞生究竟解决了哪些过去的痛点：

### 1. 依赖版本冲突与管理 {#1}

在大型项目中，手动管理众多库的版本很容易掉坑，不同库之间版本不兼容会导致 ClassNotFound 或 NoSuchMethod 错误。Starter 通常由 Spring 官方或第三方维护者出品，他们在 Starter 的依赖里**锁定了兼容的版本组合** （SpringBoot 本身还有一个版本对齐的 BOM 管理所有 Starter 依赖版本）。这意味着只要我们选定了一个 SpringBoot 版本，对应的官方 Starters 都有合理的依赖版本，不用自己纠结选哪个版本的驱动、哪版客户端库，这**极大减少了版本冲突的风险**。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
      </dependencies>
    </dependencyManagement>
              
### 2. 繁杂配置的标准化 {#2}

没有 Starter 时，不同第三方组件的整合各有各的配置方法，项目中可能散落着 XML、properties、Java Config，多处修改才能完成一个组件的接入。这种配置分散不仅增加了初始集成难度，也给后期维护带来麻烦（要记得修改多处地方）。

Starter 则**提供了统一的引入方式** ：约定大多数配置通过应用的 `application.properties/yml` 来集中管理，尽量减少散弹枪式的配置方式。例如，引入 `spring-boot-starter-redis` 后，你只需在 application.yml 配置 Redis 的地址和少量参数，Starter 已经让 Redis 客户端和Template等Bean完成自动注入，比起手动编写@Configuration去创建Jedis连接工厂之类要省事得多。**配置集中、约定统一**提升了可维护性。

### 3. 开发效率和学习成本 {#3}

Starter 让常用技术栈变得**傻瓜化** 。新手可能不知道"我要用Web服务需要哪些依赖？需要配置什么Servlet？"，但他只要知道选择 `spring-boot-starter-web` 就够了。Starter **隐藏了复杂性** ，提供了**友好的学习曲线** ：开发者可以先用默认配置跑通功能，然后再逐步了解如何自定义。没有 Starter 的年代，搭建环境本身就可能耗费大量时间和精力，现在这些都由 Starter 替我们做了**底层重活累活**。

可以说，Starter 促进了**Spring生态的一键集成**，降低了各项技术的上手门槛。

### 4. 约定优于配置的贯彻 {#4}

Spring Boot 崇尚**约定优于配置** ，Starter 是这种哲学的具体体现。它通过**约定好的依赖和默认行为** 让我们"省心"。例如 Spring Boot **约定** 了常用框架的默认端口、默认编码、默认日志级别等等，这些约定很多是通过各 Starter 的自动配置实现的。如果没有 Starter，每个项目可能各自为政，开发者自己去配置这些参数。而 Starter 保证了大家不做特别配置时就能有一致的行为，这对团队协作和开源项目来说都是很重要的（减少"它在我电脑上跑不通"的情况）。因此，Starter 的出现也**减少了配置错误的可能**，把最佳实践内置在框架中。

*** ** * ** ***

总之，Spring Boot Starter 之所以"香"，正因为它**极大地简化了依赖管理与配置工作**，解决了过去开发中常见的痛点，让我们更聚焦于业务本身。Starter 带来的这一系列好处，正是 SpringBoot 能流行的一个关键原因。

SpringBoot 2.x 与 3.x Starter 机制对比 {#spring-boot-2-x-3-x-starter}
----------------------------------------------------------------

Spring Boot 3.x 相对于 2.x 对 Starter（主要是自动配置加载机制）做出了一些改进和变化。理解这些差异有助于我们在不同版本下编写和使用 Starter。下面我们来对比一下：

### 1. 自动配置声明方式改变 {#1}

在 Spring Boot **2.x** ，如果你创建一个自定义 Starter，需要在其 `META-INF/spring.factories` 文件中声明自动配置类，例如：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.MyAutoConfiguration
              
Spring Boot 启动时会通过 `EnableAutoConfiguration` 键读取到你的自动配置类路径。这种方式虽然有效，但缺点是所有自动配置都堆在一个文件里，不同类型的扩展点都用同一个 spring.factories，管理上不够灵活。
> 所有类型的扩展点都挤在一个 `spring.factories` 文件里（比如监听器、环境后处理器、自定义配置等），不利于分类管理。

在 **Spring Boot 3.x** ，官方**移除了** 旧的 `spring.factories` 用于自动配置的用法，取而代之的是在 `META-INF/spring/` 目录下按类型放置**专门的 imports 文件**。对于自动配置类，应该创建文件：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    ​
    # 以下为文件具体内容
    com.example.MyFeatureAutoConfiguration
    com.example.AnotherAutoConfiguration
              
文件内容就是要自动配置的类名列表，每行一个。Spring Boot 会自动加载这个文件里列出的配置类。比如，先前 spring.factories 声明的 `com.example.MyAutoConfiguration`，现在只需在 `.AutoConfiguration.imports` 文件里写上一行 `com.example.MyAutoConfiguration` 即可，格式更简单直观。

**注意** ：如果你没有把自动配置类写到这个 imports 文件中，Spring Boot 是不会主动发现它的，即使你用了新的 `@AutoConfiguration` 注解。因为 Spring Boot 默认不会去组件扫描自动配置类，只有通过约定的位置读取。

这一改变使模块化支持更好，启动时只读取必要的配置，性能也提升了。对于**迁移**自定义 Starter 的开发者来说，需要将原先 spring.factories 里的配置类搬到新的文件中，并确保模块路径正确。

### 2. 兼容性与过渡 {#2}

Spring Boot 3.x 为了平滑过渡，对某些场景下仍保留了 `spring.factories` 的支持（比如一些非自动配置的扩展点仍可用旧机制），但已**标记为过时** 并计划移除。实际上在 Spring Boot 2.7 开始，官方文档就已提示 `spring.factories` 方式将被弃用，鼓励使用新方式。因此，如果你在 Spring Boot 3 上开发 Starter，**务必采用新写法** ；如果维护旧版 Starter，需要为 3.x 发布新版本或提供兼容方案（例如同时提供 `.imports` 和 `spring.factories` 两套配置以适配不同版本）。新旧机制的核心逻辑类似，只是**配置文件路径和格式变化**较大，一定要留意。

*** ** * ** ***

简而言之，**SpringBoot 3.x 简化和优化了 Starter 的自动装配注册机制**。对于使用者而言差别不大------依然是引入依赖即可；但对于 Starter 开发者，需要调整配置声明方式。在项目升级时也要注意引入的第三方 Starter 是否兼容 SpringBoot 3，如果不兼容可能需要升级到其新版本。

常用的 SpringBoot 官方 Starters 列表 {#spring-boot-starters}
-----------------------------------------------------

Spring Boot 官方提供了覆盖广泛领域的 Starter 家族，方便我们快速集成各种能力。下面列举一些**常用的官方 Starter**（按功能分类），以及它们提供的功能简述：

### 1. Web 应用相关 {#1-web}

*  
  `spring-boot-starter-web`：用于构建传统 Servlet Web 应用的入门依赖，包含了 Spring MVC 和嵌入式 Tomcat 容器等。引入它即可快速开发 RESTful API 和网页应用。

<!-- -->

*  
  `spring-boot-starter-webflux`：用于构建反应式 Web 应用的 Starter，包含 Spring WebFlux 和默认的 Reactor Netty 容器。适合需要高并发、非阻塞IO的场景。

<!-- -->

*  
  `spring-boot-starter-thymeleaf`：前端模板引擎 Thymeleaf 的 Starter，用于渲染服务端 HTML 页面。

<!-- -->

*  
  `spring-boot-starter-websocket`：WebSocket 实时通信支持的 Starter，包含 Spring WebSocket 等组件。

### 2. 数据存储与访问 {#2}

*  
  `spring-boot-starter-data-jpa`：面向关系型数据库的 JPA封装 Starter，包含 Spring Data JPA、Hibernate，以及默认的数据库连接池 HikariCP 等。让我们可以方便地使用基于 JPA 的持久层。

<!-- -->

*  
  `spring-boot-starter-jdbc`：基于 JDBC 直接访问数据库的 Starter，默认集成了 HikariCP 连接池。适合不需要ORM、直接用JDBC的场景。

<!-- -->

*  
  `spring-boot-starter-data-redis`：提供 Redis 键值数据库操作支持的 Starter，包含 Spring Data Redis 和 Lettuce 驱动等。

<!-- -->

*  
  `spring-boot-starter-data-mongodb`： 提供 MongoDB 文档数据库支持的 Starter，包含 Spring Data MongoDB 驱动。此外还有针对 Elasticsearch、Cassandra、Neo4j 等的 Starter，都在命名上遵循类似规则，一看便知用途。

### 3. 常用功能框架 {#3}

*  
  `spring-boot-starter-security`：Spring Security 安全框架的入门依赖，包含用于安全认证和授权的核心库。引入后默认会为应用启用基本的安全配置（如简单的登录表单），可进一步自定义。

<!-- -->

*  
  `spring-boot-starter-cache`：Spring 缓存抽象的 Starter，引入后可快速使用注解方式实现方法级缓存。

<!-- -->

*  
  `spring-boot-starter-validation`：提供 Hibernate Validator 校验框架支持的 Starter，用于参数校验等。

### 4. 监控与运维 {#4}

*  
  `spring-boot-starter-actuator`：**Spring Boot Actuator** 的 Starter，引入后提供了一系列**生产级别监控与管理功能**，如应用健康检查、指标度量、环境信息、Thread Dump 等端点。这是线上监控运维的利器。

<!-- -->

*  
  `spring-boot-starter-mail`：提供 JavaMail 邮件发送功能的 Starter。

*** ** * ** ***

以上只是冰山一角。几乎所有 Spring 家族的项目以及常见技术（比如 **Spring Batch、Spring Integration、Spring AMQP (RabbitMQ)** 等等）官方都提供了对应的 Starter。在命名上，**官方 Starter 统一以** `spring-boot-starter-` 前缀开头，方便识别。

需要注意的是，**第三方社区提供的 Starter** 通常不会以 `spring-boot-starter` 开头，以免与官方命名冲突。比如著名的 ElasticSearch 搜索引擎有社区提供的 Starter 叫做 `elasticsearch-spring-boot-starter`，而不是 `spring-boot-starter-elasticsearch`（后者其实是官方提供的 Spring Data Elasticsearch 的 Starter）。

总之，通过合理选择和组合 Starters，我们能像搭积木一样快捷搭建出功能完整的应用。

最小可运行的自定义 Starter 示例 {#starter}
-------------------------------

纸上得来终觉浅，最后我们通过一个**简单的自定义 Starter** 实例，来实际看看如何编写和使用一个 Starter。假设我们要封装一个**自定义的 HelloService** ，它提供一个 `sayHello()` 方法返回问候语。我们希望通过 Starter 来自动配置这个服务，并让使用者开箱即用地获得 HelloService。下面是项目的大致结构：

自定义 Starter 示例项目结构：以上展示的是一个名为 `hello-spring-boot-starter` 的 Maven 工程结构。可以看到，在 `src/main/java` 下我们有需要自动配置的组件类，例如 HelloService 及其配置类 HelloAutoConfiguration；在 `src/main/resources` 下的 META-INF 目录，我们放置了 `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件以注册自动配置类。整个 Starter 最终打包发布为一个 Jar，供其他项目引入。

![iShot_2025-06-26_20.53.57.png](https://article-images.zsxq.com/Flq6VrXrDY-rv-_He7kDwJPsKYYK)

### 1. 核心功能代码 {#1}

首先，我们编写 HelloService 及其自动配置类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 提供简单业务功能的服务类
    public class HelloService {
        public String sayHello() {
            return "Breathe：向你问好！";
        }
    }
              
HelloAutoConfiguration 自动装配类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 自动配置类，将 HelloService 注册为 Bean
    @Configuration  // 标明这是配置类
    @ConditionalOnMissingBean(HelloService.class)  // 当容器中没有 HelloService 时才生效
    public class HelloAutoConfiguration {
    ​
        @Bean
        public HelloService helloService() {
            // 可以在这里定制 HelloService，比如读取配置属性来调整行为
            return new HelloService();
        }
    }
              
这里我们用了 `@ConditionalOnMissingBean`，确保如果用户自己已经提供了一个名为 helloService 的Bean，我们的默认配置就不重复注册，保持 SpringBoot 一贯的可扩展性。另外，在实际场景中，我们可能会为 HelloService 提供可配置的属性（比如问候语内容可配置）。这时可以引入 `@ConfigurationProperties` 注解的属性类，并在自动配置时通过 `@EnableConfigurationProperties` 注册它，从而让用户在 application.yml 里配置参数。例如我们可以有：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Data
    @ConfigurationProperties(prefix="hello")
    public class HelloProperties {
        private String message = "Hello, Spring Boot Starter!";
    }
    ​
    @Configuration
    @EnableConfigurationProperties(HelloProperties.class)
    public class HelloAutoConfiguration {
    ​
        @Bean
        @ConditionalOnMissingBean
        public HelloService helloService(HelloProperties props) {
            return new HelloService(props.getMessage());
        }
    }
              
如此用户可在配置文件里通过 `hello.message=你好，Starter` 来定制消息。但为了简洁，我们的示例暂时不展开属性配置的细节。

### 2. 注册自动配置 {#2}

有了上面的自动配置类，接下来需要让 Spring Boot 在启动时知道它的存在。在 **Spring Boot 2.x** 中，我们会在 `META-INF/spring.factories` 写入：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.starter.HelloAutoConfiguration
              
在 **Spring Boot 3.x** 中，则应该在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件里加入配置类全名，如：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    com.example.starter.HelloAutoConfiguration 
              
把这一行添加到 `.imports` 文件后，我们的 Starter 打包发布时就携带了自动配置声明。Spring Boot 应用在引入该 Starter 后，会扫描到这条声明并据此加载 HelloAutoConfiguration。

### 3. 发布与依赖声明 {#3}

当我们的 Starter 编写完毕并发布到仓库后，其他人要使用就非常简单了------就和使用官方 Starter 差不多。在他们的 SpringBoot 项目的 **pom.xml** 中引入依赖：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>com.example</groupId>
        <artifactId>hello-spring-boot-starter</artifactId>
        <version>1.0.0</version>
    </dependency>
              
就这一行依赖，立刻就**触发了 Starter 的优势**：应用启动时自动注册了 HelloService。
> 实际的坐标应以大家发布到仓库的为准。

文末总结
----

通过这篇文章，我们解读了什么是 Spring Boot Starter，以及它背后的原理和用途。从"拎包入住"的便利性，到底层自动装配的工作流程和新老版本的差异，我们进行了较为全面的探索。希望这些内容能帮助大家更好地理解 SpringBoot 是如何**优雅地管理依赖和配置**的，以及在需要扩展框架功能时该如何着手编写自己的 Starter。

当你下次使用 SpringBoot 时，不妨留意一下那些熟悉的 Starters ------ 正是它们在背后默默做着"管家"，让我们的开发旅程更加顺畅。正如 Starter 名字所蕴含的，它是**启动**应用的基石，也是连接 Spring Boot 丰富生态的纽带。愿大家在掌握 Starter 原理后，能够更加游刃有余地驾驭 SpringBoot，写出优雅高效的代码！

完结，撒花 🎉  
什么是SpringBoot-Starter？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：SpringBoot Starter 就像开发者的贴心"料理包"------一行依赖，配料齐全，配置就绪，让你从此告别东拼西凑，专注业务本味。

课程目录如下所示：

*  
引子：SpringBoot 的"小帮手"  
*  
Starter 核心原理揭秘  
*  
为什么会有 Starter？解决了哪些问题？  
*  
SpringBoot 2.x 与 3.x Starter 机制对比  
*  
常用的 SpringBoot 官方 Starters 列表  
*  
最小可运行的自定义 Starter 示例  
*  
  文末总结

引子：SpringBoot 的"小帮手" {#spring-boot}
-----------------------------------

试想一下，在传统开发中我们要使用某个框架功能（比如数据库访问或消息队列），常常需要**东拼西凑** 地添加一堆依赖，还要配置各种参数。这就像下厨做菜前，还得满世界找食材、调料，步骤繁琐又容易出错。而 **Spring Boot Starter** （启动器）就像一个贴心的"大厨助手"或 **预先配好的料理包** ，帮我们一次性备齐所需"食材"（依赖）和默认配置，让开发者**开箱即用，少操很多心** 。Spring Boot Starter 用轻松的话来说，就是 Spring Boot 世界里的"小帮手"，只要把它引入项目，它就会**自动帮你准备好**相关技术栈需要的各种依赖，并且偷偷帮你做好配置工作。结果就是，你只需要专注于业务逻辑，许多底层繁琐的配置都被默默安排妥当了。

举个例子，之前如果想要让项目具备 Web 服务能力，需要引入以下依赖。你开始迷茫：**这么多依赖，我到底应该用哪个？为什么我需要记住这么多东西**？
事实上，在 Spring Boot 出现之前，这种"选择困难症"确实困扰了很多人，包括我。正经人谁会记那么多依赖地址呢。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.x.x.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.x.x.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.x.x.RELEASE</version>
    </dependency>
              
为了解决这个问题，Spring 团队给出了一个非常人性化的方案：**Spring Boot Starter**。

用大白话讲，Starter 就像是一个礼盒。里面装好了你想要的所有东西（依赖和配置），你只需要一个依赖，整个功能全都给你安排妥当。想用 Web 功能吗？加上一个依赖就行了：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
              
发现什么了吗？没错，甚至连版本号都不需要写，因为 SpringBoot 已经帮你统一管理了。

Starter 背景与作用 {#starter}
------------------------

在没有 SpringBoot 的年代，引入第三方组件通常意味着要在 Maven/Gradle 里添加多个坐标依赖，然后再去查文档写大量配置文件，非常麻烦**（依赖版本不匹配、配置分散各处都是常见痛点）** 。SpringBoot 提出了"约定优于配置"的理念，Starters 正是这一理念的产物。**Starter 的作用**简单来说有以下几点：

### 1. 依赖管理一站式 {#1}

一个 Starter 包含了一组相关的依赖，就像"自助套餐"。你加上一个 Starter，相当于引入了一系列经过官方测试搭配好的依赖，避免了自己挨个寻找版本兼容的库。例如，要使用 Spring Data JPA 访问数据库，只需添加 `spring-boot-starter-data-jpa`，它会自动引入 Spring Data JPA、Hibernate 以及数据库连接池等必要组件。不再需要东翻西找依赖，一个 Starter 就**帮你打包好了所有常用库**。

### 2. 自动配置减负 {#2}

Starter **配合 Spring Boot 的自动装配** 机制，能够根据类路径上的依赖自动进行默认配置。这意味着许多以前需要写的样板配置代码，现在 Spring Boot 会替你完成。例如，引入 `spring-boot-starter-web` 后，由于类路径有 Spring MVC 和 Tomcat，应用启动时会自动配置好 **DispatcherServlet、嵌入式 Tomcat 容器等** ，开发者无需再写XML或@Bean去配置这些。Starter **大大减少了配置分散的问题**，将繁琐的配置集中在框架内部约定好，让我们专注于少量必要的属性调整即可。

### 3. 开箱即用的默认 {#3}

大多数 Starter 都提供了一套合理的默认行为。例如 `spring-boot-starter-logging` 会默认使用 Logback 日志框架，`spring-boot-starter-web` 默认使用 Tomcat 容器并开启 Spring MVC。这些默认配置遵循官方**最佳实践** ，避免了开发者从零开始配置。同时如果默认不符合要求，我们仍可以通过 application.properties/yaml 来微调参数，Spring Boot 会自动将配置绑定到对应组件。可以说，Starter **实现了"约定优于配置"**：在有合理默认的前提下，减少开发者的选择成本和配置工作量。

*** ** * ** ***

概括来说，Spring Boot Starter 的出现**解决了依赖管理杂乱和配置碎片化的问题** 。过去可能我们需要拷贝粘贴各种依赖坐标、写很多配置，现在只需引入一个 Starter，**绝大部分配置就帮你默默搞定**。这也是为什么 Spring Boot 能让项目搭建变得如此简单快速的关键原因之一。

Starter 核心原理揭秘 {#starter}
-------------------------

Starters 之所以能做到"引入即用"，背后离不开 **Spring Boot 自动装配（Auto-Configuration）机制** 的支撑。下面让我们以拟人化的方式，跟随 Spring Boot 的启动流程，看看 Starter 是如何把相应功能**自动注入**到应用中的：

![iShot_2025-06-26_20.53.56.png](https://article-images.zsxq.com/Fjfuo6rZGCbc_rKpq65QIYEbWpOH)

当我们在主应用类上标注了 `@SpringBootApplication`（其内部含有`@EnableAutoConfiguration`注解）时，Spring Boot 会在启动时做如下事情：

### 1. 搜集候选自动配置类 {#1}

Spring Boot 利用 `SpringFactoriesLoader` 去扫描应用类路径下所有 JAR 包里的特殊配置文件。对于 Spring Boot 2.x，这个文件就是每个 Starter JAR 中 `META-INF/spring.factories`；而在 Spring Boot 3.x，这个机制有所改变，变成在每个 Starter JAR 中查找 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。这两个文件本质上都是列出该 Starter 提供的**自动配置类列表** 。比如在 Spring Boot 2.x，一个 Starter 可能在 `spring.factories` 中声明：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.MyFeatureAutoConfiguration
              
Spring Boot 读取到这个条目，就知道这个 Starter 包含 `MyFeatureAutoConfiguration` 这个自动配置类需要被加载。而在 Spring Boot 3.x，我们不再使用键值对配置，而是直接在 `AutoConfiguration.imports` 文件里逐行列出自动配置类全限定名，例如：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    com.example.MyFeatureAutoConfiguration
    com.example.AnotherAutoConfiguration
              
Spring Boot 3.x 会扫描 `META-INF/spring/*.imports` 目录下的文件来收集自动配置类名单。这一新机制更简单直观，每行一个类名，不需要再像旧格式那样写键值对，性能和模块化也有所提升。**无论新旧版本**，此步骤的结果都是：根据所有引入的 Starter，Spring Boot 得到了一张"需要尝试自动配置的配置类清单"。

### 2. 按条件装配自动配置类 {#2}

拿到所有候选的自动配置类名后（可能成百上千个），SpringBoot 并不会傻乎乎地把它们统统装配进来，而是会挨个筛选，**判断条件是否满足** 。这得益于 SpringBoot 提供的一系列**条件注解**，常用的如：

*  
`@ConditionalOnClass`：当类路径下存在某个类时才生效。  
*  
`@ConditionalOnMissingClass`：缺少某类时生效。  
*  
`@ConditionalOnBean`：当容器中已有某个 Bean 时/不在时生效。  
*  
  `@ConditionalOnProperty`：当配置文件中某个属性有指定值时生效等等。

这些条件注解被广泛地应用在各个自动配置类上，用于细粒度地控制配置是否需要自动装配。

其中最重要的一个条件是 `@ConditionalOnMissingBean`。它表示"仅当容器中**没有某个特定类型的 Bean** 时，才执行自动配置"。这确保了**当开发者自己定义了同类型的 Bean 时，自动配置会"知趣地"退让**，不会再创建重复的默认Bean。

例如，SpringBoot 会在 JDBC 数据源的自动配置类中用 `@ConditionalOnMissingBean(DataSource.class)` 判断，如果你已经手动提供了一个 `DataSource` Bean，那么默认的自动配置 DataSource 就不会再生效。这种机制保证了 Starter 提供默认配置的同时，不会妨碍你去覆盖/自定义配置。
> **小科普** ：SpringBoot 为了提升启动效率，甚至会在应用启动早期就利用注解处理器预先生成一份关于条件注解的元数据 (`META-INF/spring-autoconfigure-metadata.properties`)。这样在真正逐个评估条件前，可以快速跳过一些明显不满足条件的自动配置，从而加快启动。总之，条件装配机制让自动配置更加智能和高效。

### 3. 注册 Bean 到容器 {#3-bean}

通过条件筛选后，符合条件的自动配置类会被实例化并发挥作用。其实每个自动配置类本质上就是一个加了 `@Configuration` 注解的普通 Spring 配置类，其中定义了一系列 `@Bean` 方法用于注册组件。例如，`WebMvcAutoConfiguration` 会注册 DispatcherServlet、HandlerMapping、HttpMessageConverters 等 Web 开发需要的组件；`DataSourceAutoConfiguration` 会创建数据源连接池 Bean 等等。一旦这些配置类被加载，其内部定义的 Bean 就会按照 Spring 容器的规则被注册。

至此，原本我们需要手工配置的许多 Bean，因为 Starter 的引入而**在后台自动完成注册**了。

*** ** * ** ***

综上所述，SpringBoot Starter 能够自动注入所需功能，**完全是托管于 Spring Boot 自动装配机制** 。Starters 本身通常并不包含太多代码（很多官方 Starter 甚至只是一个 POM 依赖集合），但它们的存在触发了 SpringBoot 去加载对应的 AutoConfiguration **配置模块**。

总之，**Starter 和自动装配是一对好搭档**：Starter 提供依赖和入口，自动装配提供智能配置逻辑，两者结合使得 SpringBoot 应用具有开箱即用的特性。

为什么会有 Starter？解决了哪些问题？ {#starter}
---------------------------------

通过上面的介绍，相信大家已经体会到 Starter 带来的诸多便利。这里我们再**总结提升**一下，Spring Boot Starter 的诞生究竟解决了哪些过去的痛点：

### 1. 依赖版本冲突与管理 {#1}

在大型项目中，手动管理众多库的版本很容易掉坑，不同库之间版本不兼容会导致 ClassNotFound 或 NoSuchMethod 错误。Starter 通常由 Spring 官方或第三方维护者出品，他们在 Starter 的依赖里**锁定了兼容的版本组合** （SpringBoot 本身还有一个版本对齐的 BOM 管理所有 Starter 依赖版本）。这意味着只要我们选定了一个 SpringBoot 版本，对应的官方 Starters 都有合理的依赖版本，不用自己纠结选哪个版本的驱动、哪版客户端库，这**极大减少了版本冲突的风险**。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
      </dependencies>
    </dependencyManagement>
              
### 2. 繁杂配置的标准化 {#2}

没有 Starter 时，不同第三方组件的整合各有各的配置方法，项目中可能散落着 XML、properties、Java Config，多处修改才能完成一个组件的接入。这种配置分散不仅增加了初始集成难度，也给后期维护带来麻烦（要记得修改多处地方）。

Starter 则**提供了统一的引入方式** ：约定大多数配置通过应用的 `application.properties/yml` 来集中管理，尽量减少散弹枪式的配置方式。例如，引入 `spring-boot-starter-redis` 后，你只需在 application.yml 配置 Redis 的地址和少量参数，Starter 已经让 Redis 客户端和Template等Bean完成自动注入，比起手动编写@Configuration去创建Jedis连接工厂之类要省事得多。**配置集中、约定统一**提升了可维护性。

### 3. 开发效率和学习成本 {#3}

Starter 让常用技术栈变得**傻瓜化** 。新手可能不知道"我要用Web服务需要哪些依赖？需要配置什么Servlet？"，但他只要知道选择 `spring-boot-starter-web` 就够了。Starter **隐藏了复杂性** ，提供了**友好的学习曲线** ：开发者可以先用默认配置跑通功能，然后再逐步了解如何自定义。没有 Starter 的年代，搭建环境本身就可能耗费大量时间和精力，现在这些都由 Starter 替我们做了**底层重活累活**。

可以说，Starter 促进了**Spring生态的一键集成**，降低了各项技术的上手门槛。

### 4. 约定优于配置的贯彻 {#4}

Spring Boot 崇尚**约定优于配置** ，Starter 是这种哲学的具体体现。它通过**约定好的依赖和默认行为** 让我们"省心"。例如 Spring Boot **约定** 了常用框架的默认端口、默认编码、默认日志级别等等，这些约定很多是通过各 Starter 的自动配置实现的。如果没有 Starter，每个项目可能各自为政，开发者自己去配置这些参数。而 Starter 保证了大家不做特别配置时就能有一致的行为，这对团队协作和开源项目来说都是很重要的（减少"它在我电脑上跑不通"的情况）。因此，Starter 的出现也**减少了配置错误的可能**，把最佳实践内置在框架中。

*** ** * ** ***

总之，Spring Boot Starter 之所以"香"，正因为它**极大地简化了依赖管理与配置工作**，解决了过去开发中常见的痛点，让我们更聚焦于业务本身。Starter 带来的这一系列好处，正是 SpringBoot 能流行的一个关键原因。

SpringBoot 2.x 与 3.x Starter 机制对比 {#spring-boot-2-x-3-x-starter}
----------------------------------------------------------------

Spring Boot 3.x 相对于 2.x 对 Starter（主要是自动配置加载机制）做出了一些改进和变化。理解这些差异有助于我们在不同版本下编写和使用 Starter。下面我们来对比一下：

### 1. 自动配置声明方式改变 {#1}

在 Spring Boot **2.x** ，如果你创建一个自定义 Starter，需要在其 `META-INF/spring.factories` 文件中声明自动配置类，例如：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.MyAutoConfiguration
              
Spring Boot 启动时会通过 `EnableAutoConfiguration` 键读取到你的自动配置类路径。这种方式虽然有效，但缺点是所有自动配置都堆在一个文件里，不同类型的扩展点都用同一个 spring.factories，管理上不够灵活。
> 所有类型的扩展点都挤在一个 `spring.factories` 文件里（比如监听器、环境后处理器、自定义配置等），不利于分类管理。

在 **Spring Boot 3.x** ，官方**移除了** 旧的 `spring.factories` 用于自动配置的用法，取而代之的是在 `META-INF/spring/` 目录下按类型放置**专门的 imports 文件**。对于自动配置类，应该创建文件：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    ​
    # 以下为文件具体内容
    com.example.MyFeatureAutoConfiguration
    com.example.AnotherAutoConfiguration
              
文件内容就是要自动配置的类名列表，每行一个。Spring Boot 会自动加载这个文件里列出的配置类。比如，先前 spring.factories 声明的 `com.example.MyAutoConfiguration`，现在只需在 `.AutoConfiguration.imports` 文件里写上一行 `com.example.MyAutoConfiguration` 即可，格式更简单直观。

**注意** ：如果你没有把自动配置类写到这个 imports 文件中，Spring Boot 是不会主动发现它的，即使你用了新的 `@AutoConfiguration` 注解。因为 Spring Boot 默认不会去组件扫描自动配置类，只有通过约定的位置读取。

这一改变使模块化支持更好，启动时只读取必要的配置，性能也提升了。对于**迁移**自定义 Starter 的开发者来说，需要将原先 spring.factories 里的配置类搬到新的文件中，并确保模块路径正确。

### 2. 兼容性与过渡 {#2}

Spring Boot 3.x 为了平滑过渡，对某些场景下仍保留了 `spring.factories` 的支持（比如一些非自动配置的扩展点仍可用旧机制），但已**标记为过时** 并计划移除。实际上在 Spring Boot 2.7 开始，官方文档就已提示 `spring.factories` 方式将被弃用，鼓励使用新方式。因此，如果你在 Spring Boot 3 上开发 Starter，**务必采用新写法** ；如果维护旧版 Starter，需要为 3.x 发布新版本或提供兼容方案（例如同时提供 `.imports` 和 `spring.factories` 两套配置以适配不同版本）。新旧机制的核心逻辑类似，只是**配置文件路径和格式变化**较大，一定要留意。

*** ** * ** ***

简而言之，**SpringBoot 3.x 简化和优化了 Starter 的自动装配注册机制**。对于使用者而言差别不大------依然是引入依赖即可；但对于 Starter 开发者，需要调整配置声明方式。在项目升级时也要注意引入的第三方 Starter 是否兼容 SpringBoot 3，如果不兼容可能需要升级到其新版本。

常用的 SpringBoot 官方 Starters 列表 {#spring-boot-starters}
-----------------------------------------------------

Spring Boot 官方提供了覆盖广泛领域的 Starter 家族，方便我们快速集成各种能力。下面列举一些**常用的官方 Starter**（按功能分类），以及它们提供的功能简述：

### 1. Web 应用相关 {#1-web}

*  
  `spring-boot-starter-web`：用于构建传统 Servlet Web 应用的入门依赖，包含了 Spring MVC 和嵌入式 Tomcat 容器等。引入它即可快速开发 RESTful API 和网页应用。

<!-- -->

*  
  `spring-boot-starter-webflux`：用于构建反应式 Web 应用的 Starter，包含 Spring WebFlux 和默认的 Reactor Netty 容器。适合需要高并发、非阻塞IO的场景。

<!-- -->

*  
  `spring-boot-starter-thymeleaf`：前端模板引擎 Thymeleaf 的 Starter，用于渲染服务端 HTML 页面。

<!-- -->

*  
  `spring-boot-starter-websocket`：WebSocket 实时通信支持的 Starter，包含 Spring WebSocket 等组件。

### 2. 数据存储与访问 {#2}

*  
  `spring-boot-starter-data-jpa`：面向关系型数据库的 JPA封装 Starter，包含 Spring Data JPA、Hibernate，以及默认的数据库连接池 HikariCP 等。让我们可以方便地使用基于 JPA 的持久层。

<!-- -->

*  
  `spring-boot-starter-jdbc`：基于 JDBC 直接访问数据库的 Starter，默认集成了 HikariCP 连接池。适合不需要ORM、直接用JDBC的场景。

<!-- -->

*  
  `spring-boot-starter-data-redis`：提供 Redis 键值数据库操作支持的 Starter，包含 Spring Data Redis 和 Lettuce 驱动等。

<!-- -->

*  
  `spring-boot-starter-data-mongodb`： 提供 MongoDB 文档数据库支持的 Starter，包含 Spring Data MongoDB 驱动。此外还有针对 Elasticsearch、Cassandra、Neo4j 等的 Starter，都在命名上遵循类似规则，一看便知用途。

### 3. 常用功能框架 {#3}

*  
  `spring-boot-starter-security`：Spring Security 安全框架的入门依赖，包含用于安全认证和授权的核心库。引入后默认会为应用启用基本的安全配置（如简单的登录表单），可进一步自定义。

<!-- -->

*  
  `spring-boot-starter-cache`：Spring 缓存抽象的 Starter，引入后可快速使用注解方式实现方法级缓存。

<!-- -->

*  
  `spring-boot-starter-validation`：提供 Hibernate Validator 校验框架支持的 Starter，用于参数校验等。

### 4. 监控与运维 {#4}

*  
  `spring-boot-starter-actuator`：**Spring Boot Actuator** 的 Starter，引入后提供了一系列**生产级别监控与管理功能**，如应用健康检查、指标度量、环境信息、Thread Dump 等端点。这是线上监控运维的利器。

<!-- -->

*  
  `spring-boot-starter-mail`：提供 JavaMail 邮件发送功能的 Starter。

*** ** * ** ***

以上只是冰山一角。几乎所有 Spring 家族的项目以及常见技术（比如 **Spring Batch、Spring Integration、Spring AMQP (RabbitMQ)** 等等）官方都提供了对应的 Starter。在命名上，**官方 Starter 统一以** `spring-boot-starter-` 前缀开头，方便识别。

需要注意的是，**第三方社区提供的 Starter** 通常不会以 `spring-boot-starter` 开头，以免与官方命名冲突。比如著名的 ElasticSearch 搜索引擎有社区提供的 Starter 叫做 `elasticsearch-spring-boot-starter`，而不是 `spring-boot-starter-elasticsearch`（后者其实是官方提供的 Spring Data Elasticsearch 的 Starter）。

总之，通过合理选择和组合 Starters，我们能像搭积木一样快捷搭建出功能完整的应用。

最小可运行的自定义 Starter 示例 {#starter}
-------------------------------

纸上得来终觉浅，最后我们通过一个**简单的自定义 Starter** 实例，来实际看看如何编写和使用一个 Starter。假设我们要封装一个**自定义的 HelloService** ，它提供一个 `sayHello()` 方法返回问候语。我们希望通过 Starter 来自动配置这个服务，并让使用者开箱即用地获得 HelloService。下面是项目的大致结构：

自定义 Starter 示例项目结构：以上展示的是一个名为 `hello-spring-boot-starter` 的 Maven 工程结构。可以看到，在 `src/main/java` 下我们有需要自动配置的组件类，例如 HelloService 及其配置类 HelloAutoConfiguration；在 `src/main/resources` 下的 META-INF 目录，我们放置了 `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件以注册自动配置类。整个 Starter 最终打包发布为一个 Jar，供其他项目引入。

![iShot_2025-06-26_20.53.57.png](https://article-images.zsxq.com/Flq6VrXrDY-rv-_He7kDwJPsKYYK)

### 1. 核心功能代码 {#1}

首先，我们编写 HelloService 及其自动配置类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 提供简单业务功能的服务类
    public class HelloService {
        public String sayHello() {
            return "Breathe：向你问好！";
        }
    }
              
HelloAutoConfiguration 自动装配类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 自动配置类，将 HelloService 注册为 Bean
    @Configuration  // 标明这是配置类
    @ConditionalOnMissingBean(HelloService.class)  // 当容器中没有 HelloService 时才生效
    public class HelloAutoConfiguration {
    ​
        @Bean
        public HelloService helloService() {
            // 可以在这里定制 HelloService，比如读取配置属性来调整行为
            return new HelloService();
        }
    }
              
这里我们用了 `@ConditionalOnMissingBean`，确保如果用户自己已经提供了一个名为 helloService 的Bean，我们的默认配置就不重复注册，保持 SpringBoot 一贯的可扩展性。另外，在实际场景中，我们可能会为 HelloService 提供可配置的属性（比如问候语内容可配置）。这时可以引入 `@ConfigurationProperties` 注解的属性类，并在自动配置时通过 `@EnableConfigurationProperties` 注册它，从而让用户在 application.yml 里配置参数。例如我们可以有：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Data
    @ConfigurationProperties(prefix="hello")
    public class HelloProperties {
        private String message = "Hello, Spring Boot Starter!";
    }
    ​
    @Configuration
    @EnableConfigurationProperties(HelloProperties.class)
    public class HelloAutoConfiguration {
    ​
        @Bean
        @ConditionalOnMissingBean
        public HelloService helloService(HelloProperties props) {
            return new HelloService(props.getMessage());
        }
    }
              
如此用户可在配置文件里通过 `hello.message=你好，Starter` 来定制消息。但为了简洁，我们的示例暂时不展开属性配置的细节。

### 2. 注册自动配置 {#2}

有了上面的自动配置类，接下来需要让 Spring Boot 在启动时知道它的存在。在 **Spring Boot 2.x** 中，我们会在 `META-INF/spring.factories` 写入：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.starter.HelloAutoConfiguration
              
在 **Spring Boot 3.x** 中，则应该在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件里加入配置类全名，如：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    com.example.starter.HelloAutoConfiguration 
              
把这一行添加到 `.imports` 文件后，我们的 Starter 打包发布时就携带了自动配置声明。Spring Boot 应用在引入该 Starter 后，会扫描到这条声明并据此加载 HelloAutoConfiguration。

### 3. 发布与依赖声明 {#3}

当我们的 Starter 编写完毕并发布到仓库后，其他人要使用就非常简单了------就和使用官方 Starter 差不多。在他们的 SpringBoot 项目的 **pom.xml** 中引入依赖：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>com.example</groupId>
        <artifactId>hello-spring-boot-starter</artifactId>
        <version>1.0.0</version>
    </dependency>
              
就这一行依赖，立刻就**触发了 Starter 的优势**：应用启动时自动注册了 HelloService。
> 实际的坐标应以大家发布到仓库的为准。

文末总结
----

通过这篇文章，我们解读了什么是 Spring Boot Starter，以及它背后的原理和用途。从"拎包入住"的便利性，到底层自动装配的工作流程和新老版本的差异，我们进行了较为全面的探索。希望这些内容能帮助大家更好地理解 SpringBoot 是如何**优雅地管理依赖和配置**的，以及在需要扩展框架功能时该如何着手编写自己的 Starter。

当你下次使用 SpringBoot 时，不妨留意一下那些熟悉的 Starters ------ 正是它们在背后默默做着"管家"，让我们的开发旅程更加顺畅。正如 Starter 名字所蕴含的，它是**启动**应用的基石，也是连接 Spring Boot 丰富生态的纽带。愿大家在掌握 Starter 原理后，能够更加游刃有余地驾驭 SpringBoot，写出优雅高效的代码！

完结，撒花 🎉

