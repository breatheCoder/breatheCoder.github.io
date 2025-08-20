2025年06月27日 08:30  
快速启动oneThread后端项目，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节将指导大家完成后端项目的本地克隆，熟悉各功能模块，完成 oneThread 依赖的配置中心接入等，并启动相关后端模块。

课程目录如下所示：

*  
依赖环境  
*  
项目下载安装  
*  
服务列表  
*  
  启动 oneThread 项目

依赖环境
----

### 1. IDE {#1-ide}

IntelliJ IDEA 2023+

如果大家没有破解版的 IDEA 2023 及以上，社区版本也未尝不可。社区版本和默认版本在一些功能和插件支持上会有阉割，但是实测下来问题并不大。

IDEA 下载地址：<https://www.jetbrains.com/zh-cn/idea>

![image-20250626104253212.png](https://article-images.zsxq.com/FsIQvLp_z7CQBsD-vpbXsqlYhPAv "image-20250626104253212.png")

点击下载按钮，并下载下方的社区免费版本。

![image-20250626104515776.png](https://article-images.zsxq.com/FgQxj5Zng3AjOP_MYk06vA3VS_lL "image-20250626104515776.png")

### 2. JDK {#2-jdk}

oneThread 系统框架底层依赖 SpringBoot3，而这个版本对 JDK 的要求最低是 17。所以，我们需要将项目的 JDK 修改为 17 版本，避免项目编译或运行报错。

*  
[Azul Zulu Windows](https://www.azul.com/downloads/?version=java-17-lts&os=windows&package=jdk#zulu)  
*  
  [Azul Zulu MacOS](https://www.azul.com/downloads/?version=java-17-lts&os=macos&package=jdk#zulu)

### 3. Maven {#3-maven}

我们可以先使用 IDEA 默认的 Maven 尝试编译项目，如果成功就不用做下述任何配置。如果失败，请参考下述内容尝试更换 Maven 版本。
> 因为 Maven 版本过低 3.6.x 或者 3.9.3 及以上版本，引起和 IDEA 的冲突等问题，导致大家项目编译报错。

![image-20250626104122700.png](https://article-images.zsxq.com/FpGixFUEYrYNx8F85cYIg8AZsWtn "image-20250626104122700.png")

项目下载安装
------

### 1. 克隆项目 {#1}

打开 Gitcode 项目地址：<https://gitcode.net/nageoffer/onethread> 复制对应的 SSH 或 HTTP 克隆地址。
> 如果访问项目地址返回项目不存在或者 404，是因为没有权限。访问该地址申请项目权限：<https://t.zsxq.com/ziDKV>

不要图省事选择下载 ZIP，因为下载后的项目是没办法通过 Git 去更新远程仓库最新代码的。oneThread 代码后续可能还会不断更新迭代，每次打开项目都可以选择 Pull 下最新代码。

![image-20250626095846013.png](https://article-images.zsxq.com/Frz1T8XgOQTMAWAtV5urbt2IuT3Q "image-20250626095846013.png")

打开 IntelliJ IDEA，菜单栏顶部找到 Git -\> Clone 选项。不同电脑 Windows 或者 Mac 的位置可能有所不同，找到 Clone 这个按钮即可。

![image-20250626100046470.png](https://article-images.zsxq.com/FkP3xe3jwaxB6tMB6hkrsn6hyoul "image-20250626100046470.png")

URL 文本框填写 oneThread 的 HTTP 或者 SSH 地址，比如 SSH 的地址：`git@gitcode.net:nageoffer/onethread.git`，Directory 填写项目存储在本地的目录地址。

![image-20250626100745906.png](https://article-images.zsxq.com/FghZW_TO2iQXgqmYmnatcwxddfQ2 "image-20250626100745906.png")

等待克隆及 Maven 初始化即可。

还有一点值得说的是，因为项目后期可能增加功能或者进行问题修复等代码更新场景，所以建议大家打开项目学习前，先更新下最新代码。

![image-20250626104834399.png](https://article-images.zsxq.com/Fm0pJIOCGikdvzQsKmm5mv1ImBsa "image-20250626104834399.png")

### 2. 配置环境 {#2}

IDE 中打开 `Project Structure...` 配置，查看 JDK 选项。

![image-20250626105525126.png](https://article-images.zsxq.com/FgOGIesGEtC9bw1H1SoQfBaxKHsw "image-20250626105525126.png")

检查 JDK 版本是否正确，如果不是 JDK17 项目会编译报错，同时 Maven 打包不正确。

![image-20250626105652803.png](https://article-images.zsxq.com/FlEwAvZPd35ps2soZpezj9zFJQjR "image-20250626105652803.png")

接下来，可在项目右侧目录点击 Maven 图标测试是否具备运行环境。如果 Maven 打包运行没问题后，代表环境已经测试通过，接下来就可以启动项目学习了。

![image-20250626110054910.png](https://article-images.zsxq.com/Fn9YPtEENJeFEON5kuujgdOe0lsh "image-20250626110054910.png")

服务列表
----

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    ├── core # 动态线程池核心模块包，实现动态线程池相关基础类定义
    ├── dashboard-dev # 应广大同学要求，以后每个项目尽量有前端页面方便查看和调试
    ├── example # 动态线程池示例包，演示线程池动态参数变更、监控和告警等功能
    │   ├── apollo-example
    │   └── nacos-cloud-example
    ├── spring-base # 动态线程池基础模块包，包含Spring扫描动态线程池、是否启用以及Banner打印等
    └── starter # 动态线程池配置中心组件包，实现线程池结合Spring框架和配置中心动态刷新
        ├── adapter # 动态线程池适配层，比如对接 Web 容器 Tomcat 线程池等
        │   └── web-spring-boot-starter # Web 容器线程池组件库
        ├── apollo-spring-boot-starter # Apollo 配置中心动态监控线程池组件库
        ├── common-spring-boot-starter # 配置中心公共监听等逻辑抽象组件库
        ├── dashboard-dev-spring-boot-starter # 控制台 API 组件库
        └── nacos-cloud-spring-boot-starter # Nacos 配置中心动态监控线程池组件库
              
启动 oneThread 项目 {#one-thread}
-----------------------------

### 1. 启动后端服务 {#1}

咱们这里直接使用星球通用云服务器 Nacos，如果大家本地有 Nacos 的话，VM 参数配置中跳过有 Nacos 的即可。

在 Idea 中打开关于应用程序启动配置页面，配置相关的 VM 参数。

![image-20250626222629845.png](https://article-images.zsxq.com/FmI1qqbmZ60Pp89nAB37J0sVvaLq "image-20250626222629845.png")

如果大家刚拉下来项目，有可能是没有服务列表的，大家将上图两个服务的应用程序类启动下即可。
> 报错也无所谓，只要能出现配置类执行下述流程就好。

打开编辑应用配置程序之后，点击 `Modify options` 添加 `Add VM options` 选项，将 VM 参数选项打开即可。

![image-20250626224743687.png](https://article-images.zsxq.com/Fkuglw9l4g7bo8HyuAE1ToP1G044 "image-20250626224743687.png")

`NacosCloudExampleApplication` 添加以下 VM 参数到新增加的 `VM options` 输入框中。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    -Dunique-name=-ding-ma
    -Dspring.cloud.nacos.server-addr=common-nacos-dev.magestack.cn:8848
    --add-opens=java.base/java.util.concurrent=ALL-UNNAMED
              
`DashboardDevApplication` 添加以下 VM 参数到新增加的 `VM options` 输入框中，点击 OK 即可完成启动。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    -Donethread.nacos.server-addr=common-nacos-dev.magestack.cn:8848
              
Q：`unique-name` 参数是什么？

A：我们通过 Nacos 配置中心读取各自对应的配置，如果不使用 `unique-name` 区分，大家的 配置名称是一致的，那么就会出现配置读取错乱的情况。各自改个不会和别人冲突的，比如自己名称英文加数字之类的。

Q：`--add-opens` 参数是什么？

A：在 Java 9+ 中，模块需要显式声明它们导出了哪些包(`exports`)以及开放哪些包允许深度反射(`opens`)。`java.base` 默认只 `exports` 了它的公共API（如 `java.util.concurrent` 包的公共类和方法），但**不会自动`opens`任何包** 。这里放开 JUC 包下的权限，方便后续业务进行。

### 2. Nacos 创建配置文件 {#2-nacos}

访问 [Nacos 控制台](http://common-nacos-dev.magestack.cn:8848/nacos/index.html) 创建动态线程池配置文件。

![image-20250626230145354.png](https://article-images.zsxq.com/FjhJznwUprwtsanbRJk8d0NJgbFR "image-20250626230145354.png")

项目中文件地址：`example/nacos-cloud-example/src/main/resources/nacos-config.yaml`

将项目中的 Nacos 配置放到下述配置内容框中，将其中的几个配置参数进行替换，其中 `receives` 字段改为自己的手机号。

![image-20250629202920130.png](https://article-images.zsxq.com/Fnm1kRcFCKn1qlfFyCLRYsTvp-bd)

因为通用云服务器大家创建的配置比较多，所以可能点击发布后，找不到自己文件的情况。这种情况下，通过页面的 Data ID 筛选框筛选即可。

然后依次启动 `NacosCloudExampleApplication` 和 `DashboardDevApplication` 服务即可。

### 3. 验证动态线程池效果 {#3}

在 Nacos 配置文件中变更线程池的相关参数，如果 `NacosCloudExampleApplication` 控制台日志能够正常打印，即为启动成功。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    2025-06-26T23:42:34.712+08:00  INFO 51192 --- [resher-thread_0] c.c.s.r.DynamicThreadPoolRefreshListener : [onethread-producer] Dynamic thread pool parameter changed:
        corePoolSize: 12 => 24
        maximumPoolSize: 24 => 36
        capacity: 10000 => 10000
        keepAliveTime: 19999 => 19999
        rejectedType: CallerRunsPolicy => CallerRunsPolicy
        allowCoreThreadTimeOut: false => false
              
完结，撒花 🎉  
快速启动oneThread后端项目，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节将指导大家完成后端项目的本地克隆，熟悉各功能模块，完成 oneThread 依赖的配置中心接入等，并启动相关后端模块。

课程目录如下所示：

*  
依赖环境  
*  
项目下载安装  
*  
服务列表  
*  
  启动 oneThread 项目

依赖环境
----

### 1. IDE {#1-ide}

IntelliJ IDEA 2023+

如果大家没有破解版的 IDEA 2023 及以上，社区版本也未尝不可。社区版本和默认版本在一些功能和插件支持上会有阉割，但是实测下来问题并不大。

IDEA 下载地址：<https://www.jetbrains.com/zh-cn/idea>

![image-20250626104253212.png](https://article-images.zsxq.com/FsIQvLp_z7CQBsD-vpbXsqlYhPAv "image-20250626104253212.png")

点击下载按钮，并下载下方的社区免费版本。

![image-20250626104515776.png](https://article-images.zsxq.com/FgQxj5Zng3AjOP_MYk06vA3VS_lL "image-20250626104515776.png")

### 2. JDK {#2-jdk}

oneThread 系统框架底层依赖 SpringBoot3，而这个版本对 JDK 的要求最低是 17。所以，我们需要将项目的 JDK 修改为 17 版本，避免项目编译或运行报错。

*  
[Azul Zulu Windows](https://www.azul.com/downloads/?version=java-17-lts&os=windows&package=jdk#zulu)  
*  
  [Azul Zulu MacOS](https://www.azul.com/downloads/?version=java-17-lts&os=macos&package=jdk#zulu)

### 3. Maven {#3-maven}

我们可以先使用 IDEA 默认的 Maven 尝试编译项目，如果成功就不用做下述任何配置。如果失败，请参考下述内容尝试更换 Maven 版本。
> 因为 Maven 版本过低 3.6.x 或者 3.9.3 及以上版本，引起和 IDEA 的冲突等问题，导致大家项目编译报错。

![image-20250626104122700.png](https://article-images.zsxq.com/FpGixFUEYrYNx8F85cYIg8AZsWtn "image-20250626104122700.png")

项目下载安装
------

### 1. 克隆项目 {#1}

打开 Gitcode 项目地址：<https://gitcode.net/nageoffer/onethread> 复制对应的 SSH 或 HTTP 克隆地址。
> 如果访问项目地址返回项目不存在或者 404，是因为没有权限。访问该地址申请项目权限：<https://t.zsxq.com/ziDKV>

不要图省事选择下载 ZIP，因为下载后的项目是没办法通过 Git 去更新远程仓库最新代码的。oneThread 代码后续可能还会不断更新迭代，每次打开项目都可以选择 Pull 下最新代码。

![image-20250626095846013.png](https://article-images.zsxq.com/Frz1T8XgOQTMAWAtV5urbt2IuT3Q "image-20250626095846013.png")

打开 IntelliJ IDEA，菜单栏顶部找到 Git -\> Clone 选项。不同电脑 Windows 或者 Mac 的位置可能有所不同，找到 Clone 这个按钮即可。

![image-20250626100046470.png](https://article-images.zsxq.com/FkP3xe3jwaxB6tMB6hkrsn6hyoul "image-20250626100046470.png")

URL 文本框填写 oneThread 的 HTTP 或者 SSH 地址，比如 SSH 的地址：`git@gitcode.net:nageoffer/onethread.git`，Directory 填写项目存储在本地的目录地址。

![image-20250626100745906.png](https://article-images.zsxq.com/FghZW_TO2iQXgqmYmnatcwxddfQ2 "image-20250626100745906.png")

等待克隆及 Maven 初始化即可。

还有一点值得说的是，因为项目后期可能增加功能或者进行问题修复等代码更新场景，所以建议大家打开项目学习前，先更新下最新代码。

![image-20250626104834399.png](https://article-images.zsxq.com/Fm0pJIOCGikdvzQsKmm5mv1ImBsa "image-20250626104834399.png")

### 2. 配置环境 {#2}

IDE 中打开 `Project Structure...` 配置，查看 JDK 选项。

![image-20250626105525126.png](https://article-images.zsxq.com/FgOGIesGEtC9bw1H1SoQfBaxKHsw "image-20250626105525126.png")

检查 JDK 版本是否正确，如果不是 JDK17 项目会编译报错，同时 Maven 打包不正确。

![image-20250626105652803.png](https://article-images.zsxq.com/FlEwAvZPd35ps2soZpezj9zFJQjR "image-20250626105652803.png")

接下来，可在项目右侧目录点击 Maven 图标测试是否具备运行环境。如果 Maven 打包运行没问题后，代表环境已经测试通过，接下来就可以启动项目学习了。

![image-20250626110054910.png](https://article-images.zsxq.com/Fn9YPtEENJeFEON5kuujgdOe0lsh "image-20250626110054910.png")

服务列表
----

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    ├── core # 动态线程池核心模块包，实现动态线程池相关基础类定义
    ├── dashboard-dev # 应广大同学要求，以后每个项目尽量有前端页面方便查看和调试
    ├── example # 动态线程池示例包，演示线程池动态参数变更、监控和告警等功能
    │   ├── apollo-example
    │   └── nacos-cloud-example
    ├── spring-base # 动态线程池基础模块包，包含Spring扫描动态线程池、是否启用以及Banner打印等
    └── starter # 动态线程池配置中心组件包，实现线程池结合Spring框架和配置中心动态刷新
        ├── adapter # 动态线程池适配层，比如对接 Web 容器 Tomcat 线程池等
        │   └── web-spring-boot-starter # Web 容器线程池组件库
        ├── apollo-spring-boot-starter # Apollo 配置中心动态监控线程池组件库
        ├── common-spring-boot-starter # 配置中心公共监听等逻辑抽象组件库
        ├── dashboard-dev-spring-boot-starter # 控制台 API 组件库
        └── nacos-cloud-spring-boot-starter # Nacos 配置中心动态监控线程池组件库
              
启动 oneThread 项目 {#one-thread}
-----------------------------

### 1. 启动后端服务 {#1}

咱们这里直接使用星球通用云服务器 Nacos，如果大家本地有 Nacos 的话，VM 参数配置中跳过有 Nacos 的即可。

在 Idea 中打开关于应用程序启动配置页面，配置相关的 VM 参数。

![image-20250626222629845.png](https://article-images.zsxq.com/FmI1qqbmZ60Pp89nAB37J0sVvaLq "image-20250626222629845.png")

如果大家刚拉下来项目，有可能是没有服务列表的，大家将上图两个服务的应用程序类启动下即可。
> 报错也无所谓，只要能出现配置类执行下述流程就好。

打开编辑应用配置程序之后，点击 `Modify options` 添加 `Add VM options` 选项，将 VM 参数选项打开即可。

![image-20250626224743687.png](https://article-images.zsxq.com/Fkuglw9l4g7bo8HyuAE1ToP1G044 "image-20250626224743687.png")

`NacosCloudExampleApplication` 添加以下 VM 参数到新增加的 `VM options` 输入框中。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    -Dunique-name=-ding-ma
    -Dspring.cloud.nacos.server-addr=common-nacos-dev.magestack.cn:8848
    --add-opens=java.base/java.util.concurrent=ALL-UNNAMED
              
`DashboardDevApplication` 添加以下 VM 参数到新增加的 `VM options` 输入框中，点击 OK 即可完成启动。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    -Donethread.nacos.server-addr=common-nacos-dev.magestack.cn:8848
              
Q：`unique-name` 参数是什么？

A：我们通过 Nacos 配置中心读取各自对应的配置，如果不使用 `unique-name` 区分，大家的 配置名称是一致的，那么就会出现配置读取错乱的情况。各自改个不会和别人冲突的，比如自己名称英文加数字之类的。

Q：`--add-opens` 参数是什么？

A：在 Java 9+ 中，模块需要显式声明它们导出了哪些包(`exports`)以及开放哪些包允许深度反射(`opens`)。`java.base` 默认只 `exports` 了它的公共API（如 `java.util.concurrent` 包的公共类和方法），但**不会自动`opens`任何包** 。这里放开 JUC 包下的权限，方便后续业务进行。

### 2. Nacos 创建配置文件 {#2-nacos}

访问 [Nacos 控制台](http://common-nacos-dev.magestack.cn:8848/nacos/index.html) 创建动态线程池配置文件。

![image-20250626230145354.png](https://article-images.zsxq.com/FjhJznwUprwtsanbRJk8d0NJgbFR "image-20250626230145354.png")

项目中文件地址：`example/nacos-cloud-example/src/main/resources/nacos-config.yaml`

将项目中的 Nacos 配置放到下述配置内容框中，将其中的几个配置参数进行替换，其中 `receives` 字段改为自己的手机号。

![image-20250629202920130.png](https://article-images.zsxq.com/Fnm1kRcFCKn1qlfFyCLRYsTvp-bd)

因为通用云服务器大家创建的配置比较多，所以可能点击发布后，找不到自己文件的情况。这种情况下，通过页面的 Data ID 筛选框筛选即可。

然后依次启动 `NacosCloudExampleApplication` 和 `DashboardDevApplication` 服务即可。

### 3. 验证动态线程池效果 {#3}

在 Nacos 配置文件中变更线程池的相关参数，如果 `NacosCloudExampleApplication` 控制台日志能够正常打印，即为启动成功。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    2025-06-26T23:42:34.712+08:00  INFO 51192 --- [resher-thread_0] c.c.s.r.DynamicThreadPoolRefreshListener : [onethread-producer] Dynamic thread pool parameter changed:
        corePoolSize: 12 => 24
        maximumPoolSize: 24 => 36
        capacity: 10000 => 10000
        keepAliveTime: 19999 => 19999
        rejectedType: CallerRunsPolicy => CallerRunsPolicy
        allowCoreThreadTimeOut: false => false
              
完结，撒花 🎉

