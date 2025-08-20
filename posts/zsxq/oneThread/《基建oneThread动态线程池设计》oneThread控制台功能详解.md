20250629173344855.png](https://article-images.zsxq.com/FgqqrtpFQTgPJ5_ledL2LIKVl4xK "image-20250629173344855.png")

用户名和密码配置在 dashboard-dev 模块的 `application.yaml` 中修改：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      users: # 避免再使用数据库，用户名直接固定写到配置文件
        - admin,admin
        - test,test

oneThread 项目中使用了 SaToken 框架来实现登录功能，一款非常优秀的开源框架。由于我们并未使用 Redis 或 JWT 等持久化存储方案，因此在每次项目重启后，登录状态会丢失，用户需要重新登录。

项目列表
----

由于 Nacos 本身并没有"项目"的概念，我们通过在配置文件中增加一个扩展参数来实现项目的划分。

![image-20250629210153340.png](https://article-images.zsxq.com/FjpQksKoUeap4xnbp9rp66RqFYE- "image-20250629210153340.png")

项目列表前端展示如下所示：

![image-20250629210732120.png](https://article-images.zsxq.com/FjlyC5TvZgkkjMyc5MEWoQiKJ7LU "image-20250629210732120.png")

列表字段说明：

*  
**命名空间** ：对应 Nacos 中的命名空间，可理解为项目组的标识。  
*  
**服务名** ：系统名称，可视作该项目组下的具体项目。  
*  
**实例数量** ：表示该项目在 Nacos 注册中心注册的实例数量。例如，本地仅启动一个实例时，显示为 1；若启动两个实例，则显示为 2。  
*  
**线程池数量** ：配置文件中定义的线程池实例数量。  
*  
  **Web线程池** ：标识配置文件中是否包含 Web 线程池配置；若包含，则支持 Web 线程池的动态调整。

线程池管理
-----

### 1. 线程池列表 {#1}

当前页面展示所有命名空间和服务中包含的线程池配置，也是登录后的默认页面。页面可以向右滑动，因为配置较多，所以才用了滑动方式展示。

![image-20250629212551220.png](https://article-images.zsxq.com/FlelC7k0WOF-ljWkLRemH2og-2BY "image-20250629212551220.png")

列表字段说明：

*  
**命名空间/服务名称** ：含义同上，已在前文说明。  
*  
**线程池标识** ：对应线程池配置项 `onethread.executors[x].thread-pool-id` 的值。  
*  
**核心线程数等参数** ：以下各项为线程池的基础配置参数，此处不再赘述。  
*  
  **实例数量** ：当前线程池关联的已启动服务实例数量。

编辑功能：

![image-20250629213354688.png](https://article-images.zsxq.com/FkZBLt_Dxs7e4WMj5FBGUw-PvM1h "image-20250629213354688.png")

若修改上述参数，变更请求会通过 `dashboard-dev` 服务组装参数并调用 Nacos 接口，更新对应的配置文件。各客户端应用通过监听 Nacos 配置中心，可实现线程池配置的实时刷新。

实例列表功能：点击可跳转至线程池实例详情页面；若实例数量为 0，则无法跳转。

### 2. 线程池实例 {#2}

该列表用于展示当前线程池在各服务实例中的实时运行参数。

![image-20250629213844775.png](https://article-images.zsxq.com/FnsK1lx0y6uaJ-eIYLZuPOnQVdU7 "image-20250629213844775.png")

查看详情功能：

![image-20250629220942760.png](https://article-images.zsxq.com/FqklWwvX9bhn6BOuB3CpyFnVzB64 "image-20250629220942760.png")

线程池所在服务中最新的全量参数展示，点击"刷新"按钮，可向线程池实例所在的应用发起请求，获取并展示最新的运行时参数。

线程池监控
-----

该页面依托 Prometheus 存储和采集线程池监控数据，并通过 Grafana 进行可视化展示。

![image-20250629214716579.png](https://article-images.zsxq.com/Fgvknr_bkmUN_LAeYGth5Owr943J "image-20250629214716579.png")

目前默认调用我封装的 Grafana 服务。如果需要使用自定义的 Grafana 实例，可通过修改 dashboard-dev 服务 `application.yaml` 文件中的 `grafana.url` 参数进行配置。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      users: # 避免再使用数据库，用户名直接固定写到配置文件
        - admin,admin
        - test,test
      nacos:
        server-addr: http://127.0.0.1:8848
      user-login:
        exclude:
          interfaces: |-
            /api/onethread-dashboard/auth/login
      namespaces: # 需要从 Nacos 中读取以下命名空间（自定义）配置文件检索动态线程池
        - public
        - framework
        - common
        - test
        - prod
      grafana:
        # 如果本地有安装 Grafana 展示，可以替换为本地路径
        url: http://grafana.nageoffer.com/d/gxBvKxYNz/7adffa3?orgId=1&from=now-6h&to=now&timezone=browser&var-application_name=nacos-cloud-example&var-dynamic_thread_pool_id=onethread-consumer&refresh=5s&theme=light&kiosk=true

Web 线程池管理 {#web}
----------------

### 1. 线程池列表 {#1}

当前页面展示所有命名空间和服务中包含的 Web 线程池配置。

![image-20250629215733509.png](https://article-images.zsxq.com/FiZgYei6sxKWGfqi0qyD5cbvZnyx "image-20250629215733509.png")

列表字段说明：

*  
**命名空间/服务名称** ：含义同上，已在前文说明。  
*  
**数据ID** ：对应 Nacos 中的 data-id 字段。  
*  
**分组标识** ：对应 Nacos 中的 group 字段。  
*  
**Web容器名称** ：见名知意。  
*  
  **实例数量** ：当前 Web 线程池关联的已启动服务实例数量。

编辑功能：

![image-20250629220151374.png](https://article-images.zsxq.com/Fquzxj0pXBjy_o7WIUoZoxgELw3g "image-20250629220151374.png")

若修改上述参数，变更请求会通过 `dashboard-dev` 服务组装参数并调用 Nacos 接口，更新对应的配置文件。各客户端应用通过监听 Nacos 配置中心，可实现 Web 线程池配置的实时刷新。

实例列表功能：点击可跳转至 Web 线程池实例详情页面；若实例数量为 0，则无法跳转。

### 2. 线程池实例 {#2}

该列表用于展示当前 Web 线程池在各服务实例中的实时运行参数。

![image-20250629220237339.png](https://article-images.zsxq.com/FgX66jYWPzlvz_SwSpFHO36up44H "image-20250629220237339.png")

查看详情功能：

![image-20250629220907385.png](https://article-images.zsxq.com/FvfElTizNTAwmeSzGXMX1xOT8Ioj "image-20250629220907385.png")

Web 线程池所在服务中最新的全量参数展示，点击"刷新"按钮，可向 Web 线程池实例所在的应用发起请求，获取并展示最新的运行时参数。

完结，撒花 🎉  
oneThread控制台功能详解，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★☆☆☆☆ - 简单  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节讲解 oneThread 控制台相关功能介绍，帮助大家更好了解如何接触控制台学习动态线程池。

课程目录如下所示：

*  
控制台设计介绍  
*  
用户登录  
*  
项目列表  
*  
线程池管理  
*  
线程池监控  
*  
  Web 线程池管理

控制台设计介绍
-------

在最早介绍 oneThread 的时候和大家聊过，oneThread 是基于 **配置中心** 构建的动态可观测 Java 线程池框架，这种设计是没有前端控制台的，所有操作围绕配置中心展开。

为了帮助大家更好理解动态线程池，oneThread 在基于配置中心的基础上，抽象了一层控制台。简单一句话说明就是，基于 Nacos 配置中心和注册中心实现的控制台。
> 大家看咱们项目结构时候，如果 module 命名后面跟着 `-dev` 就是基于控制台的个性化开发。常规公司使用基于配置中心的动态线程池是没有这块设计的。

因为 Nginx 启动和前端源码启动访问方式不同，这里再补充下：

*  
Nginx 一键启动：<http://localhost:5176>  
*  
  前端源码启动：<http://localhost:5777>

用户登录
----

默认用户名和密码是 admin / admin，点击登录按钮即可。

![image-20250629173344855.png](https://article-images.zsxq.com/FgqqrtpFQTgPJ5_ledL2LIKVl4xK "image-20250629173344855.png")

用户名和密码配置在 dashboard-dev 模块的 `application.yaml` 中修改：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      users: # 避免再使用数据库，用户名直接固定写到配置文件
        - admin,admin
        - test,test

oneThread 项目中使用了 SaToken 框架来实现登录功能，一款非常优秀的开源框架。由于我们并未使用 Redis 或 JWT 等持久化存储方案，因此在每次项目重启后，登录状态会丢失，用户需要重新登录。

项目列表
----

由于 Nacos 本身并没有"项目"的概念，我们通过在配置文件中增加一个扩展参数来实现项目的划分。

![image-20250629210153340.png](https://article-images.zsxq.com/FjpQksKoUeap4xnbp9rp66RqFYE- "image-20250629210153340.png")

项目列表前端展示如下所示：

![image-20250629210732120.png](https://article-images.zsxq.com/FjlyC5TvZgkkjMyc5MEWoQiKJ7LU "image-20250629210732120.png")

列表字段说明：

*  
**命名空间** ：对应 Nacos 中的命名空间，可理解为项目组的标识。  
*  
**服务名** ：系统名称，可视作该项目组下的具体项目。  
*  
**实例数量** ：表示该项目在 Nacos 注册中心注册的实例数量。例如，本地仅启动一个实例时，显示为 1；若启动两个实例，则显示为 2。  
*  
**线程池数量** ：配置文件中定义的线程池实例数量。  
*  
  **Web线程池** ：标识配置文件中是否包含 Web 线程池配置；若包含，则支持 Web 线程池的动态调整。

线程池管理
-----

### 1. 线程池列表 {#1}

当前页面展示所有命名空间和服务中包含的线程池配置，也是登录后的默认页面。页面可以向右滑动，因为配置较多，所以才用了滑动方式展示。

![image-20250629212551220.png](https://article-images.zsxq.com/FlelC7k0WOF-ljWkLRemH2og-2BY "image-20250629212551220.png")

列表字段说明：

*  
**命名空间/服务名称** ：含义同上，已在前文说明。  
*  
**线程池标识** ：对应线程池配置项 `onethread.executors[x].thread-pool-id` 的值。  
*  
**核心线程数等参数** ：以下各项为线程池的基础配置参数，此处不再赘述。  
*  
  **实例数量** ：当前线程池关联的已启动服务实例数量。

编辑功能：

![image-20250629213354688.png](https://article-images.zsxq.com/FkZBLt_Dxs7e4WMj5FBGUw-PvM1h "image-20250629213354688.png")

若修改上述参数，变更请求会通过 `dashboard-dev` 服务组装参数并调用 Nacos 接口，更新对应的配置文件。各客户端应用通过监听 Nacos 配置中心，可实现线程池配置的实时刷新。

实例列表功能：点击可跳转至线程池实例详情页面；若实例数量为 0，则无法跳转。

### 2. 线程池实例 {#2}

该列表用于展示当前线程池在各服务实例中的实时运行参数。

![image-20250629213844775.png](https://article-images.zsxq.com/FnsK1lx0y6uaJ-eIYLZuPOnQVdU7 "image-20250629213844775.png")

查看详情功能：

![image-20250629220942760.png](https://article-images.zsxq.com/FqklWwvX9bhn6BOuB3CpyFnVzB64 "image-20250629220942760.png")

线程池所在服务中最新的全量参数展示，点击"刷新"按钮，可向线程池实例所在的应用发起请求，获取并展示最新的运行时参数。

线程池监控
-----

该页面依托 Prometheus 存储和采集线程池监控数据，并通过 Grafana 进行可视化展示。

![image-20250629214716579.png](https://article-images.zsxq.com/Fgvknr_bkmUN_LAeYGth5Owr943J "image-20250629214716579.png")

目前默认调用我封装的 Grafana 服务。如果需要使用自定义的 Grafana 实例，可通过修改 dashboard-dev 服务 `application.yaml` 文件中的 `grafana.url` 参数进行配置。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      users: # 避免再使用数据库，用户名直接固定写到配置文件
        - admin,admin
        - test,test
      nacos:
        server-addr: http://127.0.0.1:8848
      user-login:
        exclude:
          interfaces: |-
            /api/onethread-dashboard/auth/login
      namespaces: # 需要从 Nacos 中读取以下命名空间（自定义）配置文件检索动态线程池
        - public
        - framework
        - common
        - test
        - prod
      grafana:
        # 如果本地有安装 Grafana 展示，可以替换为本地路径
        url: http://grafana.nageoffer.com/d/gxBvKxYNz/7adffa3?orgId=1&from=now-6h&to=now&timezone=browser&var-application_name=nacos-cloud-example&var-dynamic_thread_pool_id=onethread-consumer&refresh=5s&theme=light&kiosk=true

Web 线程池管理 {#web}
----------------

### 1. 线程池列表 {#1}

当前页面展示所有命名空间和服务中包含的 Web 线程池配置。

![image-20250629215733509.png](https://article-images.zsxq.com/FiZgYei6sxKWGfqi0qyD5cbvZnyx "image-20250629215733509.png")

列表字段说明：

*  
**命名空间/服务名称** ：含义同上，已在前文说明。  
*  
**数据ID** ：对应 Nacos 中的 data-id 字段。  
*  
**分组标识** ：对应 Nacos 中的 group 字段。  
*  
**Web容器名称** ：见名知意。  
*  
  **实例数量** ：当前 Web 线程池关联的已启动服务实例数量。

编辑功能：

![image-20250629220151374.png](https://article-images.zsxq.com/Fquzxj0pXBjy_o7WIUoZoxgELw3g "image-20250629220151374.png")

若修改上述参数，变更请求会通过 `dashboard-dev` 服务组装参数并调用 Nacos 接口，更新对应的配置文件。各客户端应用通过监听 Nacos 配置中心，可实现 Web 线程池配置的实时刷新。

实例列表功能：点击可跳转至 Web 线程池实例详情页面；若实例数量为 0，则无法跳转。

### 2. 线程池实例 {#2}

该列表用于展示当前 Web 线程池在各服务实例中的实时运行参数。

![image-20250629220237339.png](https://article-images.zsxq.com/FgX66jYWPzlvz_SwSpFHO36up44H "image-20250629220237339.png")

查看详情功能：

![image-20250629220907385.png](https://article-images.zsxq.com/FvfElTizNTAwmeSzGXMX1xOT8Ioj "image-20250629220907385.png")

Web 线程池所在服务中最新的全量参数展示，点击"刷新"按钮，可向 Web 线程池实例所在的应用发起请求，获取并展示最新的运行时参数。

完结，撒花 🎉

