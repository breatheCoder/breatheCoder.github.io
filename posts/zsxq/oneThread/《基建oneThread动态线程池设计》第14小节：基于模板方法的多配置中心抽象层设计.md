2025年07月21日 22:00  
基于模板方法的多配置中心抽象层设计，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本文通过对 Nacos、Apollo 两种配置中心刷新实现的对比剖析，引出其在多配置中心扩展中的设计短板，继而引入**模板方法设计模式**对刷新逻辑进行抽象重构。

课程目录如下所示：

*  
前言  
*  
多配置中心设计短板  
*  
什么是模板方法设计模式？  
*  
使用模板方法重构刷新事件  
*  
  文末总结

前言
---

在前面的章节中，为了帮助大家快速理解动态线程池的**配置刷新机制** ，我们使用了一些"临时代码"作为演示。这些代码虽然简洁直观，但并不是我们真正落地时采用的方式。

在 oneThread 的真实实现中，我们为了支持多种配置中心并保持良好的可维护性，采用了**模板方法模式** 来封装公共流程、抽象差异行为，从而实现可扩展、高内聚的动态配置刷新能力。

本文将带你完整梳理线程池配置动态化的设计演进，重点聚焦在**多配置中心支持的设计短板** 与**为何需要引入模板方法模式** 。

多配置中心设计短板
---------

目前我们已经支持 Nacos 和 Apollo 两种主流配置中心，下面是它们各自的监听实现方式。

### 1. Nacos 配置刷新事件实现 {#1-nacos}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class NacosCloudRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final ConfigService configService;
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.NacosConfig nacosConfig = properties.getNacos();
            configService.addListener(
                    nacosConfig.getDataId(),
                    nacosConfig.getGroup(),
                    new Listener() {
    ​
                        @Override
                        public Executor getExecutor() {
                            return xxx;
                        }
    ​
                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            refreshThreadPoolProperties(content); // 刷新动态线程池配置
                        }
                    });
    ​
            log.info("Dynamic thread pool refresher, add nacos cloud listener success. data-id: {}, group: {}", nacosConfig.getDataId(), nacosConfig.getGroup());
        }
    ​
        public void refreshThreadPoolProperties(String configInfo) throws IOException {
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
    ​
            BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
            log.info("Latest updated configuration: \n{}", configInfo);
            log.info("Java configuration object binding: \n{}", JSON.toJSONString(refresherProperties));
    ​
            // ...... 检测线程池参数是否变更，如果已变更则进行更新
        }
    }
              
### 2. Apollo 配置刷新事件实现 {#2-apollo}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class ApolloRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.ApolloConfig apolloConfig = properties.getApollo();
            String[] apolloNamespaces = apolloConfig.getNamespace().split(",");
    ​
            String namespace = apolloNamespaces[0];
            String configFileType = properties.getConfigFileType().getValue();
            Config config = ConfigService.getConfig(String.format("%s.%s", namespace, properties.getConfigFileType().getValue()));
    ​
            ConfigChangeListener configChangeListener = createConfigChangeListener(namespace, configFileType);
            config.addChangeListener(configChangeListener);
    ​
            log.info("Dynamic thread pool refresher, add apollo listener success. namespace: {}", namespace);
        }
    ​
        private ConfigChangeListener createConfigChangeListener(String namespace, String configFileType) {
            return configChangeEvent -> {
                // 如果 Apollo 配置文件变更，会触发该方法进行回调
                String namespaceItem = namespace.replace("." + configFileType, "");
                ConfigFileFormat configFileFormat = ConfigFileFormat.fromString(configFileType);
                ConfigFile configFile = ConfigService.getConfigFile(namespaceItem, configFileFormat);
                String content = configFile.getContent(); // 变更后的最新内容
                refreshThreadPoolProperties(content); // 刷新动态线程池配置
            };
        }
    ​
        public void refreshThreadPoolProperties(String configInfo) throws IOException {
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
    ​
            BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
            log.info("Latest updated configuration: \n{}", configInfo);
            log.info("Java configuration object binding: \n{}", JSON.toJSONString(refresherProperties));
    ​
            // ...... 检测线程池参数是否变更，如果已变更则进行更新
        }
    }
              
### 3. 存在的主要问题 {#3}

随着对配置中心的支持增加，上述实现会暴露出以下设计短板：

#### 3.1 代码重复 {#3-1}

*  
`refreshThreadPoolProperties()` 的逻辑在每个类中**重复出现** ；  
*  
`ApplicationRunner` 接口需要在每个类中**重复实现** ；  
*  
  每增加一种配置中心，都需**复制粘贴一份逻辑** ，不利于维护与扩展。

#### 3.2 行为无法约束 {#3-2}

*  
每个配置中心的监听实现类触发时机都是自由发挥，**缺乏统一约束规范** ；  
*  
容易出现调用时机不一致、监听失败无兜底、线程池变更未触发等问题；  
*  
  难以统一升级或增强公共能力（例如统一异常处理、线程池重载逻辑等）。

接下来的章节将介绍我们是如何借助**模板方法模式** 设计一个统一的刷新抽象类，并支持以最小成本扩展更多配置中心的。

什么是模板方法设计模式？
------------

### 1. 模板方法定义 {#1}

**模板方法设计模式** 是一种行为设计模式，它在一个方法中定义了一个操作的框架，而将一些步骤的实现延迟到子类中。通过这种方式，模板方法允许子类在不改变算法结构的情况下重新定义算法中的某些步骤。

通俗来讲 : 定义一个抽象类 `AbstractTemplate`，并定义一个或若干抽象方法 `abstractMethod`。

由子类去继承抽象类的同时实现抽象方法， 在抽象类的公共方法中调用抽象方法，最终调用的就是不同子类实现的方法逻辑。

### 2. 用户登录举例 {#2}

**背景：** 假设我们有一套用户登录逻辑，其总体流程如下：

* 1.  
获取用户输入；  
* 2.  
校验身份（用户名密码、验证码、OAuth token 等）；  
* 3.  
  登录成功后写入日志或发通知。

这个流程对于不同登录方式（账号密码登录、微信扫码登录、钉钉免登登录等）是一致的，只有"校验方式"不同，典型适合使用模板方法模式抽象出公共流程。

#### 2.1 抽象模板类 {#2-1}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public abstract class LoginTemplate {
    ​
        /**
         * 模板方法：定义固定登录流程
         */
        public final void login(String loginId) {
            // 1. 获取用户输入
            String input = getLoginInput(loginId);
            // 2. 执行认证逻辑（由子类决定怎么认证）
            boolean success = authenticate(input);
            if (success) {
                // 3. 登录成功后的处理
                postLogin(loginId);
            } else {
                System.out.println("[LoginTemplate] 登录失败");
            }
        }
    ​
        /**
         * 获取登录输入（可选步骤，默认实现）
         */
        protected String getLoginInput(String loginId) {
            return loginId; // 默认直接返回用户名等
        }
    ​
        /**
         * 认证逻辑（抽象方法）
         */
        protected abstract boolean authenticate(String input);
    ​
        /**
         * 登录成功后的操作（钩子方法）
         */
        protected void postLogin(String loginId) {
            System.out.println("[LoginTemplate] 登录成功，用户：" + loginId);
        }
    }
              
#### 2.2 子类一：账号密码登录 {#2-2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class UsernamePasswordLogin extends LoginTemplate {
    ​
        @Override
        protected boolean authenticate(String input) {
            System.out.println("[UsernamePasswordLogin] 正在校验用户名和密码...");
            return "admin".equals(input); // 模拟判断用户名等于 admin 才能登录
        }
    ​
        @Override
        protected void postLogin(String loginId) {
            System.out.println("[UsernamePasswordLogin] 登录成功，记录登录日志：" + loginId);
        }
    }
              
#### 2.3 子类二：微信扫码登录 {#2-3}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class WeChatScanLogin extends LoginTemplate {
    ​
        @Override
        protected boolean authenticate(String input) {
            System.out.println("[WeChatScanLogin] 正在校验微信二维码...");
            return input.startsWith("wx_"); // 模拟微信二维码前缀
        }
    ​
        @Override
        protected void postLogin(String loginId) {
            System.out.println("[WeChatScanLogin] 登录成功，推送欢迎消息：" + loginId);
        }
    }
              
#### 2.4 测试运行 {#2-4}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class LoginTest {
        public static void main(String[] args) {
            LoginTemplate login1 = new UsernamePasswordLogin();
            login1.login("admin");
    ​
            System.out.println("-----");
    ​
            LoginTemplate login2 = new WeChatScanLogin();
            login2.login("wx_123456");
        }
    }
    ​
    [UsernamePasswordLogin] 正在校验用户名和密码...
    [UsernamePasswordLogin] 登录成功，记录登录日志：admin
    -----
    [WeChatScanLogin] 正在校验微信二维码...
    [WeChatScanLogin] 登录成功，推送欢迎消息：wx_123456
              
这里拿用户名密码登录方式举例，时序图如下所示：

![image-20250721153424954.png](https://article-images.zsxq.com/Fs-AyauQ5uGzlgVbKRhBt6C9oOra "image-20250721153424954.png")

### 3. 模式优点 {#3}

*  
登录流程完全由模板类控制，流程统一；  
*  
子类只关心 **认证方式** 和 **后处理动作** ；  
*  
  方便扩展钉钉登录、验证码登录等，**无需改动已有逻辑** 。

使用模板方法重构刷新事件
------------

动态线程池在多配置中心支持的场景下，监听器注册与配置刷新存在高度相似的处理流程。为了统一逻辑、增强可扩展性，我们引入**模板方法模式** 进行重构，形成一套结构清晰、职责分明的刷新机制。

### 1. 类图 {#1}

![iShot_2025-07-19_13.07.27.png](https://article-images.zsxq.com/Fu25DFGZpgo_2HwgjWOOQmSxThyD "iShot_2025-07-19_13.07.27.png")

动态刷新场景逻辑如下：

*  
抽象类 `AbstractDynamicThreadPoolRefresher` 定义了统一的刷新流程；  
*  
子类如 `NacosCloudRefresherHandler`、`ApolloRefresherHandler` 仅需实现配置监听逻辑；  
*  
  避免重复代码，便于后续扩展其他配置中心（如 Zookeeper、ETCD、Consul）。

### 2. 时序图 {#2}

以 Nacos 为例，以下展示一次配置变更后线程池刷新的完整流程：

![image-20250721163429499.png](https://article-images.zsxq.com/Fp2bPOJHLPxQbMWJvFARGoN6eAQa "image-20250721163429499.png")

### 3. 定义抽象类 {#3}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 基于模板方法模式抽象动态线程池刷新逻辑
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-28
     */
    @Slf4j
    @RequiredArgsConstructor
    public abstract class AbstractDynamicThreadPoolRefresher implements ApplicationRunner {
    ​
        protected final BootstrapConfigProperties properties;
    ​
        /**
         * 注册配置变更监听器，由子类实现具体逻辑
         *
         * @throws Exception
         */
        protected abstract void registerListener() throws Exception;
    ​
        /**
         * 默认空实现，子类可以按需覆盖
         */
        protected void beforeRegister() {
        }
    ​
        /**
         * 默认空实现，子类可以按需覆盖
         */
        protected void afterRegister() {
        }
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            beforeRegister();
            registerListener();
            afterRegister();
        }
    ​
        @SneakyThrows
        public void refreshThreadPoolProperties(String configInfo) {
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
            BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
    ​
            // 发布线程池配置变更事件，触发所有监听器执行线程池参数对比与刷新操作
            // 当前支持的监听器包括：
            // - {@link com.nageoffer.onethread.config.common.starter.refresher.DynamicThreadPoolRefreshListener}
            // - {@link com.nageoffer.onethread.web.starter.core.WebThreadPoolRefreshListener}
            ApplicationContextHolder.publishEvent(new ThreadPoolConfigUpdateEvent(this, refresherProperties));
        }
    }
              
常见问题说明：

Q：为什么要保留 `beforeRegister` 和 `afterRegister`？

这是典型的钩子方法设计，类似 JDK 中线程池的：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ThreadPoolExecutor extends AbstractExecutorService {
    ​
        protected void beforeExecute(Thread t, Runnable r) { }
    ​
        protected void afterExecute(Runnable r, Throwable t) { }
    }
              
在注册监听器前后留有扩展点，可用于设置上下文、注入依赖或补充逻辑，提升灵活性。

Q：`ApplicationContextHolder.publishEvent` 是什么？

该方法基于 Spring 的**事件发布机制（观察者模式）** ，用于广播线程池参数的刷新事件。这是支持 Web 容器线程池刷新所必须的机制，后续章节会详细展开。

### 4. 子类 Nacos 实现 {#4-nacos}

模板方法模式下，Nacos 的刷新逻辑变得异常清晰：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * Nacos Cloud 版本刷新处理器
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-24
     */
    @Slf4j(topic = "OneThreadConfigRefresher")
    public class NacosCloudRefresherHandler extends AbstractDynamicThreadPoolRefresher {

        private ConfigService configService;

        public NacosCloudRefresherHandler(ConfigService configService, BootstrapConfigProperties properties) {
            super(properties);
            this.configService = configService;
        }

        public void registerListener() throws NacosException {
            BootstrapConfigProperties.NacosConfig nacosConfig = properties.getNacos();
            configService.addListener(
                    nacosConfig.getDataId(),
                    nacosConfig.getGroup(),
                    new Listener() {

                        @Override
                        public Executor getExecutor() {
                            return ThreadPoolExecutorBuilder.builder()
                                    .corePoolSize(1)
                                    .maximumPoolSize(1)
                                    .keepAliveTime(9999L)
                                    .workQueueType(BlockingQueueTypeEnum.SYNCHRONOUS_QUEUE)
                                    .threadFactory("clod-nacos-refresher-thread_")
                                    .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                                    .build();
                        }

                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            refreshThreadPoolProperties(configInfo);
                        }
                    });

            log.info("Dynamic thread pool refresher, add nacos cloud listener success. data-id: {}, group: {}", nacosConfig.getDataId(), nacosConfig.getGroup());
        }
    }
              
各配置中心的实现类中，如果无特殊逻辑，只需关注 `registerListener`，其余流程由抽象类统一调度。

### 5. 重构后的优势 {#5}

*  
**复用性强** ：将公共流程（如 `ApplicationRunner` 执行入口、配置解析与绑定、事件发布）抽象封装在父类中，避免子类重复实现，提升代码复用率。  
*  
**扩展性好** ：将配置中心的差异化逻辑下沉到子类中实现，新增支持只需继承抽象类并实现 `registerListener()` 方法，符合开闭原则，方便扩展和维护。  
*  
  **结构更清晰** ：登录流程集中由模板方法统一控制，配置监听与刷新职责分离，类职责单一，整体流程**可读性与可维护性大幅提升** 。

文末总结
----

在动态线程池支持多配置中心的场景下，如何实现**高复用、强扩展、低侵入** 的配置刷新机制，是架构设计中的关键考量。

本文以动态线程池在多配置中心下的刷新监听逻辑为出发点，从实际实现入手，分析了原始代码在复用性与可扩展性方面的不足。通过引入模板方法模式，将注册监听流程与配置刷新逻辑进行合理分层与抽象。

通过统一的抽象类封装注册流程与刷新逻辑，配合钩子函数与事件发布机制，实现了结构清晰、行为受控、可持续演进的动态配置刷新架构。文章并结合类图、时序图、代码示例，帮助大家完整理解该重构方案的核心价值与应用落地。

完结，撒花 🎉  
基于模板方法的多配置中心抽象层设计，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本文通过对 Nacos、Apollo 两种配置中心刷新实现的对比剖析，引出其在多配置中心扩展中的设计短板，继而引入**模板方法设计模式**对刷新逻辑进行抽象重构。

课程目录如下所示：

*  
前言  
*  
多配置中心设计短板  
*  
什么是模板方法设计模式？  
*  
使用模板方法重构刷新事件  
*  
  文末总结

前言
---

在前面的章节中，为了帮助大家快速理解动态线程池的**配置刷新机制** ，我们使用了一些"临时代码"作为演示。这些代码虽然简洁直观，但并不是我们真正落地时采用的方式。

在 oneThread 的真实实现中，我们为了支持多种配置中心并保持良好的可维护性，采用了**模板方法模式** 来封装公共流程、抽象差异行为，从而实现可扩展、高内聚的动态配置刷新能力。

本文将带你完整梳理线程池配置动态化的设计演进，重点聚焦在**多配置中心支持的设计短板** 与**为何需要引入模板方法模式** 。

多配置中心设计短板
---------

目前我们已经支持 Nacos 和 Apollo 两种主流配置中心，下面是它们各自的监听实现方式。

### 1. Nacos 配置刷新事件实现 {#1-nacos}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class NacosCloudRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final ConfigService configService;
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.NacosConfig nacosConfig = properties.getNacos();
            configService.addListener(
                    nacosConfig.getDataId(),
                    nacosConfig.getGroup(),
                    new Listener() {
    ​
                        @Override
                        public Executor getExecutor() {
                            return xxx;
                        }
    ​
                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            refreshThreadPoolProperties(content); // 刷新动态线程池配置
                        }
                    });
    ​
            log.info("Dynamic thread pool refresher, add nacos cloud listener success. data-id: {}, group: {}", nacosConfig.getDataId(), nacosConfig.getGroup());
        }
    ​
        public void refreshThreadPoolProperties(String configInfo) throws IOException {
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
    ​
            BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
            log.info("Latest updated configuration: \n{}", configInfo);
            log.info("Java configuration object binding: \n{}", JSON.toJSONString(refresherProperties));
    ​
            // ...... 检测线程池参数是否变更，如果已变更则进行更新
        }
    }
              
### 2. Apollo 配置刷新事件实现 {#2-apollo}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class ApolloRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.ApolloConfig apolloConfig = properties.getApollo();
            String[] apolloNamespaces = apolloConfig.getNamespace().split(",");
    ​
            String namespace = apolloNamespaces[0];
            String configFileType = properties.getConfigFileType().getValue();
            Config config = ConfigService.getConfig(String.format("%s.%s", namespace, properties.getConfigFileType().getValue()));
    ​
            ConfigChangeListener configChangeListener = createConfigChangeListener(namespace, configFileType);
            config.addChangeListener(configChangeListener);
    ​
            log.info("Dynamic thread pool refresher, add apollo listener success. namespace: {}", namespace);
        }
    ​
        private ConfigChangeListener createConfigChangeListener(String namespace, String configFileType) {
            return configChangeEvent -> {
                // 如果 Apollo 配置文件变更，会触发该方法进行回调
                String namespaceItem = namespace.replace("." + configFileType, "");
                ConfigFileFormat configFileFormat = ConfigFileFormat.fromString(configFileType);
                ConfigFile configFile = ConfigService.getConfigFile(namespaceItem, configFileFormat);
                String content = configFile.getContent(); // 变更后的最新内容
                refreshThreadPoolProperties(content); // 刷新动态线程池配置
            };
        }
    ​
        public void refreshThreadPoolProperties(String configInfo) throws IOException {
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
    ​
            BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
            log.info("Latest updated configuration: \n{}", configInfo);
            log.info("Java configuration object binding: \n{}", JSON.toJSONString(refresherProperties));
    ​
            // ...... 检测线程池参数是否变更，如果已变更则进行更新
        }
    }
              
### 3. 存在的主要问题 {#3}

随着对配置中心的支持增加，上述实现会暴露出以下设计短板：

#### 3.1 代码重复 {#3-1}

*  
`refreshThreadPoolProperties()` 的逻辑在每个类中**重复出现** ；  
*  
`ApplicationRunner` 接口需要在每个类中**重复实现** ；  
*  
  每增加一种配置中心，都需**复制粘贴一份逻辑** ，不利于维护与扩展。

#### 3.2 行为无法约束 {#3-2}

*  
每个配置中心的监听实现类触发时机都是自由发挥，**缺乏统一约束规范** ；  
*  
容易出现调用时机不一致、监听失败无兜底、线程池变更未触发等问题；  
*  
  难以统一升级或增强公共能力（例如统一异常处理、线程池重载逻辑等）。

接下来的章节将介绍我们是如何借助**模板方法模式** 设计一个统一的刷新抽象类，并支持以最小成本扩展更多配置中心的。

什么是模板方法设计模式？
------------

### 1. 模板方法定义 {#1}

**模板方法设计模式** 是一种行为设计模式，它在一个方法中定义了一个操作的框架，而将一些步骤的实现延迟到子类中。通过这种方式，模板方法允许子类在不改变算法结构的情况下重新定义算法中的某些步骤。

通俗来讲 : 定义一个抽象类 `AbstractTemplate`，并定义一个或若干抽象方法 `abstractMethod`。

由子类去继承抽象类的同时实现抽象方法， 在抽象类的公共方法中调用抽象方法，最终调用的就是不同子类实现的方法逻辑。

### 2. 用户登录举例 {#2}

**背景：** 假设我们有一套用户登录逻辑，其总体流程如下：

* 1.  
获取用户输入；  
* 2.  
校验身份（用户名密码、验证码、OAuth token 等）；  
* 3.  
  登录成功后写入日志或发通知。

这个流程对于不同登录方式（账号密码登录、微信扫码登录、钉钉免登登录等）是一致的，只有"校验方式"不同，典型适合使用模板方法模式抽象出公共流程。

#### 2.1 抽象模板类 {#2-1}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public abstract class LoginTemplate {
    ​
        /**
         * 模板方法：定义固定登录流程
         */
        public final void login(String loginId) {
            // 1. 获取用户输入
            String input = getLoginInput(loginId);
            // 2. 执行认证逻辑（由子类决定怎么认证）
            boolean success = authenticate(input);
            if (success) {
                // 3. 登录成功后的处理
                postLogin(loginId);
            } else {
                System.out.println("[LoginTemplate] 登录失败");
            }
        }
    ​
        /**
         * 获取登录输入（可选步骤，默认实现）
         */
        protected String getLoginInput(String loginId) {
            return loginId; // 默认直接返回用户名等
        }
    ​
        /**
         * 认证逻辑（抽象方法）
         */
        protected abstract boolean authenticate(String input);
    ​
        /**
         * 登录成功后的操作（钩子方法）
         */
        protected void postLogin(String loginId) {
            System.out.println("[LoginTemplate] 登录成功，用户：" + loginId);
        }
    }
              
#### 2.2 子类一：账号密码登录 {#2-2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class UsernamePasswordLogin extends LoginTemplate {
    ​
        @Override
        protected boolean authenticate(String input) {
            System.out.println("[UsernamePasswordLogin] 正在校验用户名和密码...");
            return "admin".equals(input); // 模拟判断用户名等于 admin 才能登录
        }
    ​
        @Override
        protected void postLogin(String loginId) {
            System.out.println("[UsernamePasswordLogin] 登录成功，记录登录日志：" + loginId);
        }
    }
              
#### 2.3 子类二：微信扫码登录 {#2-3}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class WeChatScanLogin extends LoginTemplate {
    ​
        @Override
        protected boolean authenticate(String input) {
            System.out.println("[WeChatScanLogin] 正在校验微信二维码...");
            return input.startsWith("wx_"); // 模拟微信二维码前缀
        }
    ​
        @Override
        protected void postLogin(String loginId) {
            System.out.println("[WeChatScanLogin] 登录成功，推送欢迎消息：" + loginId);
        }
    }
              
#### 2.4 测试运行 {#2-4}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class LoginTest {
        public static void main(String[] args) {
            LoginTemplate login1 = new UsernamePasswordLogin();
            login1.login("admin");
    ​
            System.out.println("-----");
    ​
            LoginTemplate login2 = new WeChatScanLogin();
            login2.login("wx_123456");
        }
    }
    ​
    [UsernamePasswordLogin] 正在校验用户名和密码...
    [UsernamePasswordLogin] 登录成功，记录登录日志：admin
    -----
    [WeChatScanLogin] 正在校验微信二维码...
    [WeChatScanLogin] 登录成功，推送欢迎消息：wx_123456
              
这里拿用户名密码登录方式举例，时序图如下所示：

![image-20250721153424954.png](https://article-images.zsxq.com/Fs-AyauQ5uGzlgVbKRhBt6C9oOra "image-20250721153424954.png")

### 3. 模式优点 {#3}

*  
登录流程完全由模板类控制，流程统一；  
*  
子类只关心 **认证方式** 和 **后处理动作** ；  
*  
  方便扩展钉钉登录、验证码登录等，**无需改动已有逻辑** 。

使用模板方法重构刷新事件
------------

动态线程池在多配置中心支持的场景下，监听器注册与配置刷新存在高度相似的处理流程。为了统一逻辑、增强可扩展性，我们引入**模板方法模式** 进行重构，形成一套结构清晰、职责分明的刷新机制。

### 1. 类图 {#1}

![iShot_2025-07-19_13.07.27.png](https://article-images.zsxq.com/Fu25DFGZpgo_2HwgjWOOQmSxThyD "iShot_2025-07-19_13.07.27.png")

动态刷新场景逻辑如下：

*  
抽象类 `AbstractDynamicThreadPoolRefresher` 定义了统一的刷新流程；  
*  
子类如 `NacosCloudRefresherHandler`、`ApolloRefresherHandler` 仅需实现配置监听逻辑；  
*  
  避免重复代码，便于后续扩展其他配置中心（如 Zookeeper、ETCD、Consul）。

### 2. 时序图 {#2}

以 Nacos 为例，以下展示一次配置变更后线程池刷新的完整流程：

![image-20250721163429499.png](https://article-images.zsxq.com/Fp2bPOJHLPxQbMWJvFARGoN6eAQa "image-20250721163429499.png")

### 3. 定义抽象类 {#3}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 基于模板方法模式抽象动态线程池刷新逻辑
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-28
     */
    @Slf4j
    @RequiredArgsConstructor
    public abstract class AbstractDynamicThreadPoolRefresher implements ApplicationRunner {
    ​
        protected final BootstrapConfigProperties properties;
    ​
        /**
         * 注册配置变更监听器，由子类实现具体逻辑
         *
         * @throws Exception
         */
        protected abstract void registerListener() throws Exception;
    ​
        /**
         * 默认空实现，子类可以按需覆盖
         */
        protected void beforeRegister() {
        }
    ​
        /**
         * 默认空实现，子类可以按需覆盖
         */
        protected void afterRegister() {
        }
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            beforeRegister();
            registerListener();
            afterRegister();
        }
    ​
        @SneakyThrows
        public void refreshThreadPoolProperties(String configInfo) {
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
            BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
    ​
            // 发布线程池配置变更事件，触发所有监听器执行线程池参数对比与刷新操作
            // 当前支持的监听器包括：
            // - {@link com.nageoffer.onethread.config.common.starter.refresher.DynamicThreadPoolRefreshListener}
            // - {@link com.nageoffer.onethread.web.starter.core.WebThreadPoolRefreshListener}
            ApplicationContextHolder.publishEvent(new ThreadPoolConfigUpdateEvent(this, refresherProperties));
        }
    }
              
常见问题说明：

Q：为什么要保留 `beforeRegister` 和 `afterRegister`？

这是典型的钩子方法设计，类似 JDK 中线程池的：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ThreadPoolExecutor extends AbstractExecutorService {
    ​
        protected void beforeExecute(Thread t, Runnable r) { }
    ​
        protected void afterExecute(Runnable r, Throwable t) { }
    }
              
在注册监听器前后留有扩展点，可用于设置上下文、注入依赖或补充逻辑，提升灵活性。

Q：`ApplicationContextHolder.publishEvent` 是什么？

该方法基于 Spring 的**事件发布机制（观察者模式）** ，用于广播线程池参数的刷新事件。这是支持 Web 容器线程池刷新所必须的机制，后续章节会详细展开。

### 4. 子类 Nacos 实现 {#4-nacos}

模板方法模式下，Nacos 的刷新逻辑变得异常清晰：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * Nacos Cloud 版本刷新处理器
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-24
     */
    @Slf4j(topic = "OneThreadConfigRefresher")
    public class NacosCloudRefresherHandler extends AbstractDynamicThreadPoolRefresher {

        private ConfigService configService;

        public NacosCloudRefresherHandler(ConfigService configService, BootstrapConfigProperties properties) {
            super(properties);
            this.configService = configService;
        }

        public void registerListener() throws NacosException {
            BootstrapConfigProperties.NacosConfig nacosConfig = properties.getNacos();
            configService.addListener(
                    nacosConfig.getDataId(),
                    nacosConfig.getGroup(),
                    new Listener() {

                        @Override
                        public Executor getExecutor() {
                            return ThreadPoolExecutorBuilder.builder()
                                    .corePoolSize(1)
                                    .maximumPoolSize(1)
                                    .keepAliveTime(9999L)
                                    .workQueueType(BlockingQueueTypeEnum.SYNCHRONOUS_QUEUE)
                                    .threadFactory("clod-nacos-refresher-thread_")
                                    .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                                    .build();
                        }

                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            refreshThreadPoolProperties(configInfo);
                        }
                    });

            log.info("Dynamic thread pool refresher, add nacos cloud listener success. data-id: {}, group: {}", nacosConfig.getDataId(), nacosConfig.getGroup());
        }
    }
              
各配置中心的实现类中，如果无特殊逻辑，只需关注 `registerListener`，其余流程由抽象类统一调度。

### 5. 重构后的优势 {#5}

*  
**复用性强** ：将公共流程（如 `ApplicationRunner` 执行入口、配置解析与绑定、事件发布）抽象封装在父类中，避免子类重复实现，提升代码复用率。  
*  
**扩展性好** ：将配置中心的差异化逻辑下沉到子类中实现，新增支持只需继承抽象类并实现 `registerListener()` 方法，符合开闭原则，方便扩展和维护。  
*  
  **结构更清晰** ：登录流程集中由模板方法统一控制，配置监听与刷新职责分离，类职责单一，整体流程**可读性与可维护性大幅提升** 。

文末总结
----

在动态线程池支持多配置中心的场景下，如何实现**高复用、强扩展、低侵入** 的配置刷新机制，是架构设计中的关键考量。

本文以动态线程池在多配置中心下的刷新监听逻辑为出发点，从实际实现入手，分析了原始代码在复用性与可扩展性方面的不足。通过引入模板方法模式，将注册监听流程与配置刷新逻辑进行合理分层与抽象。

通过统一的抽象类封装注册流程与刷新逻辑，配合钩子函数与事件发布机制，实现了结构清晰、行为受控、可持续演进的动态配置刷新架构。文章并结合类图、时序图、代码示例，帮助大家完整理解该重构方案的核心价值与应用落地。

完结，撒花 🎉

