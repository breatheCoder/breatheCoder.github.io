2025年07月29日 21:29  
Prometheus 配置 oneThread 运行时监控数据采集，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文深入介绍 oneThread 动态线程池框架与 **Prometheus 监控系统** 的集成实践，重点阐述**Prometheus 架构原理** 、**Docker 部署配置** 和**指标采集任务**的完整实现。

[Promethes 官方文档](https://prometheus.io/)

课程目录如下所示：

*  
前言  
*  
Prometheus 介绍  
*  
Docker 安装 Prometheus  
*  
Prometheus 控制台操作指南  
*  
PromQL 查询语言实践  
*  
  文末总结

前言
---

在上一篇文章中，我们详细介绍了 oneThread 框架的 Micrometer 指标监控实现。通过 Micrometer 的抽象层，我们的应用已经能够以标准格式暴露线程池监控指标。但是，要真正发挥这些指标的价值，我们还需要一个专业的监控系统来**采集、存储和分析** 这些数据。

想象一下这样的场景：
> 周一早上 9 点，你刚到公司就收到了运维同事的紧急消息："订单服务的线程池好像有问题，处理速度明显变慢了。" 在传统的监控方式下，你可能需要登录服务器查看日志，或者临时写脚本来获取线程池状态。但有了 Prometheus 监控系统，你只需要打开浏览器，访问 Prometheus 控制台，输入一个简单的查询语句：`dynamic_thread_pool_active_size{application_name="order-service"}`，立即就能看到所有线程池的活跃线程数变化趋势。

这就是 **Prometheus** 的好用之处------它不仅能自动采集应用的监控指标，还能提供强大的查询能力和历史数据分析功能。

相比传统的监控方案，Prometheus 具有以下显著优势：

*  
**Pull模式采集** ：主动从应用拉取指标，避免了推送模式的网络复杂性。  
*  
**时间序列存储** ：专门为监控数据设计的存储引擎，查询性能优异。  
*  
**PromQL查询语言** ：功能强大的查询语言，支持复杂的数据分析和聚合。  
*  
**服务发现机制** ：自动发现和监控新的服务实例。  
*  
  **告警规则引擎** ：内置告警功能，支持多种通知方式。

Prometheus 介绍 {#prometheus}
---------------------------

### 1. 什么是 Prometheus？ {#1-prometheus}

[Prometheus](https://github.com/prometheus) 是一个开源的系统监控和告警工具包，最初由 [SoundCloud](https://soundcloud.com/) 构建。自 2012 年诞生以来，许多公司和组织都采用了 Prometheus，并且该项目拥有非常活跃的开发者和用户 [社区](https://prometheus.ac.cn/community/)。它现在是一个独立的开源项目，由任何公司独立维护。为了强调这一点并明确项目的治理结构，Prometheus 于 2016 年加入了 [云原生计算基金会](https://cncf.io/)，成为继 [Kubernetes](https://kubernetes.ac.cn/) 之后第二个托管项目。

Prometheus 将其指标作为时间序列数据收集和存储，即指标信息与记录时的时间戳以及可选的键值对（称为标签）一起存储。

### 2. 特性 {#2}

Prometheus 的主要特性包括：

*  
一个多维[数据模型](https://prometheus.ac.cn/docs/concepts/data_model/)，其中时间序列数据由指标名称和键/值对标识。  
*  
PromQL，一种灵活的[查询语言](https://prometheus.ac.cn/docs/prometheus/latest/querying/basics/)，用于利用这种多维性。  
*  
不依赖分布式存储；单个服务器节点是自主的。  
*  
时间序列收集通过 HTTP 上的拉取模型进行。  
*  
通过中间网关支持[推送时间序列](https://prometheus.ac.cn/docs/instrumenting/pushing/)。  
*  
通过服务发现或静态配置发现目标。  
*  
  支持多种图形和仪表盘模式。

### 3. 什么是指标（Metric）？ {#3-metric}

指标在通俗意义上是数值测量。术语"时间序列"是指随时间记录的变化。用户希望测量的内容因应用程序而异。对于 Web 服务器，可能是请求时间；对于数据库，可能是活动连接数或活动查询数等等。

指标在理解应用程序为何以某种方式工作方面发挥着重要作用。假设您正在运行一个 Web 应用程序并发现它运行缓慢。要了解应用程序发生了什么，您需要一些信息。例如，当请求数量很高时，应用程序可能会变慢。如果您拥有请求计数指标，则可以确定原因并增加服务器数量以处理负载。

### 4. 组件 {#4}

Prometheus 生态系统由多个组件组成，其中许多是可选的

*  
主 [Prometheus 服务器](https://github.com/prometheus/prometheus)，它抓取并存储时间序列数据。  
*  
用于对应用程序代码进行埋点的[客户端库](https://prometheus.ac.cn/docs/instrumenting/clientlibs/)。  
*  
一个用于支持短生命周期作业的[推送网关](https://github.com/prometheus/pushgateway)。  
*  
用于 HAProxy、StatsD、Graphite 等服务的专用[导出器](https://prometheus.ac.cn/docs/instrumenting/exporters/)。  
*  
一个用于处理告警的[告警管理器](https://github.com/prometheus/alertmanager)。  
*  
  各种支持工具。

大多数 Prometheus 组件都是用 [Go](https://golang.ac.cn/) 编写的，这使得它们易于构建并部署为静态二进制文件。

### 5. 架构 {#5}

此图展示了 Prometheus 及其部分生态系统组件的架构：

![image-20250729212709079.png](https://article-images.zsxq.com/FsirPjnx9er499VdFCCk08Z3mzAG)Prometheus 从已埋点的作业中抓取指标，可以直接抓取，也可以通过中间推送网关抓取短生命周期作业的指标。它将所有抓取的样本存储在本地，并根据这些数据运行规则，以聚合并记录现有数据中的新时间序列或生成告警。[Grafana](https://grafana.com/) 或其他 API 消费者可用于可视化收集到的数据。

### 6. 为什么选择拉而不是推？ {#6}

Prometheus 默认采用的是一种 **"拉模型（PullModel）"** 架构，它会**主动周期性地拉取被监控目标的指标数据** 。每个被监控的服务需要暴露一个支持 **Prometheus格式** 的 HTTP 接口，通常路径是 `/metrics`（如 Spring Boot 的 `/actuator/prometheus`）。

![iShot_2025-07-28_19.28.60.png](https://article-images.zsxq.com/FqcGwgbowDLgsnRv5RbeJSgu2SPC)

通过 HTTP 进行拉取有很多优点：

*  
根据需要启动监控实例，例如在本地开发时在笔记本电脑上启动一个 Prometheus 实例进行调试。  
*  
更容易、更可靠地判断某个目标是否宕机。  
*  
  可以手动访问目标服务的指标接口，直接通过浏览器检查其健康状态。

总体而言，Prometheus 认为拉模式略优于推模式，在考虑监控系统时，拉和推不应该成为主要考虑点。

对于必须推送的情况，Prometheus 同时也提供了 [Pushgateway](https://prometheus.io/docs/instrumenting/pushing/) 组件。

Docker 安装 Prometheus {#docker-prometheus}
-----------------------------------------

### 1. 简易版安装 {#1}

考虑到部分同学对 Docker 这些命令不太熟悉，常规安装 Prometheus 需要挂载配置文件地址，同时 Windows 和 Mac、Linux 路径方式还不太一样，这里Breathe希望在 **Docker启动命令中直接传入Prometheus配置内容** ，**不挂载本地配置文件** ，实现"**一条命令部署Prometheus并用自定义配置** "。

虽然 Prometheus 官方镜像不支持直接通过命令行参数传入完整配置，但是玩了个花活实现需求。使用 `echo` + `docker run` + `--entrypoint` 动态生成配置：

* 1.  
创建一个临时容器；  
* 2.  
用 `echo` 把配置写到容器内 `/etc/prometheus/prometheus.yml`；  
* 3.  
  然后启动 Prometheus。

Docker 命令如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker run -d \
      --name prometheus \
      -p 9090:9090 \
      --entrypoint sh \
      -e TZ=Asia/Shanghai \
      prom/prometheus:v2.51.1 \
      -c 'echo "
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    ​
    scrape_configs:
      - job_name: '\''prometheus'\''
        static_configs:
          - targets: ['\''localhost:9090'\'']
    ​
      - job_name: '\''onethread-app'\''
        metrics_path: '\''/actuator/prometheus'\''
        static_configs:
          - targets: ['\''host.docker.internal:18080'\'']
        scrape_interval: 10s
        scrape_timeout: 5s
    " > /etc/prometheus/prometheus.yml && /bin/prometheus --config.file=/etc/prometheus/prometheus.yml'
              
参数说明：

*  
`--entrypoint sh`：先让容器以 shell 启动；  
*  
`-c 'echo "... && /bin/prometheus ...'`：一口气完成写入配置并启动 Prometheus；  
*  
`'\''` 是 Bash 里的转义方法，用于生成单引号 `'`；  
*  
  整体是 shell 脚本拼接实现，不需要本地文件也能运行。

***如果想简易实现oneThread动态线程Metric采集，上面这条命令足够了。上述配置文件重写等同于：`yamlglobal:scrape_interval:15s#全局采集间隔evaluation_interval:15s#告警规则评估间隔​#采集任务配置scrape_configs:#Prometheus自身监控-job_name:'prometheus'static_configs:-targets:['localhost:9090']​#oneThread应用监控-job_name:'onethread-app'metrics_path:'/actuator/prometheus'static_configs:-targets:['host.docker.internal:18080','host.docker.internal:18081']#同一任务需要采集多IP，可以逗号分割scrape_interval:10sscrape_timeout:5s`因为Docker和宿主机的IP是不一样的，如果在Docker容器中运行Prometheus时，** 要访问宿主机的服务或端口* \*，推荐使用：`host.docker.internal`。在`Linux（Dockerv20.10+）`、`macOS`和`Windows`中，Docker提供了特殊的主机名，它可以在容器内解析为宿主机的IP。`host.docker.internal:18080`表示Prometheus容器去访问宿主机18080端口的服务。###2.运行检查浏览器访问<http://localhost:9090/targets>如果可以出现下述页面，即为运行成功。![image-20250729115229461.png](https://article-images.zsxq.com/FvHA-dDQaYKQ3S7w53wd79VC48QP)可以看到咱们在Prometheus配置文件中加的两个采集任务，都在列表上展示，并且State显示为`UP`健康状态。\>大家记得把onethread-nacos-cloud-example项目运行起来，要不然上面那个任务状态会显示失败。##Prometheus控制台操作指南###1.WebUI界面概览访问`http://localhost:9090`进入PrometheusWeb控制台，主要包含以下功能模块：\*\* Graph（图表查询）\*\*：

*  
**功能** ：执行 PromQL 查询，查看指标数据和图表。  
*  
  **用途** ：数据探索、问题排查、趋势分析。

**Alerts（告警管理）** ：

*  
**功能** ：查看当前告警状态和历史记录。  
*  
  **用途** ：告警监控、规则调试。

**Status（状态信息）** ：

*  
**Targets** ：查看采集目标状态。  
*  
**ServiceDiscovery** ：查看服务发现状态。  
*  
**Configuration** ：查看当前配置。  
*  
  **Rules** ：查看告警规则状态。

### 2. Graph 页面详细操作 {#2-graph}

**基础查询操作** ：

* 1.  
  **简单指标查询** ：  
  textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                      dynamic_thread_pool_active_size
                
显示所有线程池的活跃线程数。  
* 2.  
  **标签过滤查询** ：  
  textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                      dynamic_thread_pool_active_size{application_name="xxx"}
                
只显示特定应用的线程池数据。  
* 3.  
  **时间范围选择** ：
  *  
  使用页面顶部的时间选择器。  
  *  
  支持相对时间：5m、1h、1d、7d 等。  
  *  
    支持绝对时间：指定具体的开始和结束时间。

![image-20250729152434091.png](https://article-images.zsxq.com/Fjn8EPxMjy7oRv3zeR3jr8H7pMgR)

可以点击这个 [链接](http://localhost:9090/graph?g0.expr=dynamic_thread_pool_active_size&g0.tab=0&g0.display_mode=stacked&g0.show_exemplars=0&g0.range_input=30m) 跳转到查询该规则的页面，链接上带着 URL 参数，非常方便。

**图表显示选项** ：

*  
**Graph** ：时间序列图表，显示指标随时间的变化。  
*  
**Console** ：表格形式显示当前值。  
*  
  **Table** ：以表格形式显示所有时间序列。

### 3. Configuration 页面 {#3-configuration}

**查看当前配置** ：

Status → Configuration 页面显示当前生效的完整配置，包括：

*  
**全局配置** ：采集间隔、评估间隔等。  
*  
**采集任务配置** ：所有 scrape_configs 的详细配置。  
*  
  **告警规则配置** ：rule_files 中定义的规则。

**配置热重载** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 方法1：发送 HTTP 请求
    curl -X POST http://localhost:9090/-/reload
    ​
    # 方法2：发送系统信号
    docker exec prometheus kill -HUP 1
              
PromQL 查询语言实践 {#prom-ql}
------------------------

### 1. PromQL 基础语法 {#1-prom-ql}

**即时查询** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查询当前时刻的指标值
    dynamic_thread_pool_active_size
              
**范围查询** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查询过去5分钟的指标值
    dynamic_thread_pool_active_size[5m]
              
**标签匹配** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 精确匹配
    dynamic_thread_pool_active_size{job="onethread-app"}
    ​
    # 正则匹配
    dynamic_thread_pool_active_size{application_name=~".*example.*"}
    ​
    # 不等匹配
    dynamic_thread_pool_active_size{dynamic_thread_pool_id!="onethread-producer"}
              
### 2. oneThread 指标查询实例 {#2-one-thread}

**线程池活跃度监控** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查看所有线程池的活跃线程数
    dynamic_thread_pool_active_size
    ​
    # 查看特定应用的线程池活跃度
    dynamic_thread_pool_active_size{application_name="xxx"}
    ​
    # 计算线程池使用率
    dynamic_thread_pool_active_size / dynamic_thread_pool_maximum_size * 100
              
**队列状态监控** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查看队列当前长度
    dynamic_thread_pool_queue_size
    ​
    # 计算队列使用率
    dynamic_thread_pool_queue_size / dynamic_thread_pool_queue_capacity * 100
    ​
    # 查看队列剩余容量
    dynamic_thread_pool_queue_remaining_capacity
              
**任务执行统计** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查看已完成任务数
    dynamic_thread_pool_completed_task_count
    ​
    # 计算任务完成速率（每秒）
    rate(dynamic_thread_pool_completed_task_count[5m])
    ​
    # 查看拒绝任务数
    dynamic_thread_pool_reject_count
              
### 3. 聚合函数应用 {#3}

**按应用聚合** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 计算每个应用的总活跃线程数
    sum by (application_name) (dynamic_thread_pool_active_size)
    ​
    # 计算每个应用的平均队列长度
    avg by (application_name) (dynamic_thread_pool_queue_size)
    ​
    # 查找每个应用中活跃线程数最多的线程池
    max by (application_name) (dynamic_thread_pool_active_size)
              
**时间窗口聚合** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 过去5分钟的平均活跃线程数
    avg_over_time(dynamic_thread_pool_active_size[5m])
    ​
    # 过去1小时的最大队列长度
    max_over_time(dynamic_thread_pool_queue_size[1h])
    ​
    # 过去10分钟的任务完成增长量
    increase(dynamic_thread_pool_completed_task_count[10m])
              
### 4. 复杂查询示例 {#4}

**线程池健康度评估** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 线程池压力指数（活跃线程数 + 队列长度）
    (dynamic_thread_pool_active_size + dynamic_thread_pool_queue_size) / dynamic_thread_pool_maximum_size
    ​
    # 识别高负载线程池（使用率超过80%）
    dynamic_thread_pool_active_size / dynamic_thread_pool_maximum_size > 0.8
    ​
    # 检测队列堆积严重的线程池
    dynamic_thread_pool_queue_size / dynamic_thread_pool_queue_capacity > 0.7
              
**多维度对比查询** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 对比不同线程池的性能表现
    topk(5, dynamic_thread_pool_active_size)
    ​
    # 查找最繁忙的应用
    topk(3, sum by (application_name) (dynamic_thread_pool_active_size))
    ​
    # 识别异常线程池（活跃线程数突然下降）
    dynamic_thread_pool_active_size < 
      avg_over_time(dynamic_thread_pool_active_size[1h]) * 0.5
              
### 5. PromQL 查询优化技巧 {#5-prom-ql}

**查询性能优化** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 避免：查询大时间范围的原始数据
    dynamic_thread_pool_active_size[24h]
    ​
    # 推荐：使用聚合函数减少数据量
    avg_over_time(dynamic_thread_pool_active_size[24h])
              
**标签选择优化** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 避免：使用正则匹配大量标签
    {__name__=~"dynamic_thread_pool_.*"}
    ​
    # 推荐：使用精确的标签匹配
    dynamic_thread_pool_active_size{application_name="xxx"}
              
文末总结
----

通过 Prometheus 监控系统，oneThread 框架真正实现了"可观测性"的目标。运维团队不仅能够实时了解线程池的运行状态，还能通过历史数据分析发现潜在问题，为系统优化提供数据支撑。结合前面介绍的本地日志监控和 Micrometer 指标监控，oneThread 框架已经构建了一套完整的监控体系。无论是开发调试、生产运维，还是性能分析，都能找到合适的监控工具和方法。

至此，oneThread 动态线程池框架的 Prometheus 监控集成就全部介绍完了，下一章节会带着大家一起部署 Grafana，将 Prometheus 时序数据展示在 Grafana 大屏。

完结，撒花 🎉  
Prometheus 配置 oneThread 运行时监控数据采集，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文深入介绍 oneThread 动态线程池框架与 **Prometheus 监控系统** 的集成实践，重点阐述**Prometheus 架构原理** 、**Docker 部署配置** 和**指标采集任务**的完整实现。

[Promethes 官方文档](https://prometheus.io/)

课程目录如下所示：

*  
前言  
*  
Prometheus 介绍  
*  
Docker 安装 Prometheus  
*  
Prometheus 控制台操作指南  
*  
PromQL 查询语言实践  
*  
  文末总结

前言
---

在上一篇文章中，我们详细介绍了 oneThread 框架的 Micrometer 指标监控实现。通过 Micrometer 的抽象层，我们的应用已经能够以标准格式暴露线程池监控指标。但是，要真正发挥这些指标的价值，我们还需要一个专业的监控系统来**采集、存储和分析** 这些数据。

想象一下这样的场景：
> 周一早上 9 点，你刚到公司就收到了运维同事的紧急消息："订单服务的线程池好像有问题，处理速度明显变慢了。" 在传统的监控方式下，你可能需要登录服务器查看日志，或者临时写脚本来获取线程池状态。但有了 Prometheus 监控系统，你只需要打开浏览器，访问 Prometheus 控制台，输入一个简单的查询语句：`dynamic_thread_pool_active_size{application_name="order-service"}`，立即就能看到所有线程池的活跃线程数变化趋势。

这就是 **Prometheus** 的好用之处------它不仅能自动采集应用的监控指标，还能提供强大的查询能力和历史数据分析功能。

相比传统的监控方案，Prometheus 具有以下显著优势：

*  
**Pull模式采集** ：主动从应用拉取指标，避免了推送模式的网络复杂性。  
*  
**时间序列存储** ：专门为监控数据设计的存储引擎，查询性能优异。  
*  
**PromQL查询语言** ：功能强大的查询语言，支持复杂的数据分析和聚合。  
*  
**服务发现机制** ：自动发现和监控新的服务实例。  
*  
  **告警规则引擎** ：内置告警功能，支持多种通知方式。

Prometheus 介绍 {#prometheus}
---------------------------

### 1. 什么是 Prometheus？ {#1-prometheus}

[Prometheus](https://github.com/prometheus) 是一个开源的系统监控和告警工具包，最初由 [SoundCloud](https://soundcloud.com/) 构建。自 2012 年诞生以来，许多公司和组织都采用了 Prometheus，并且该项目拥有非常活跃的开发者和用户 [社区](https://prometheus.ac.cn/community/)。它现在是一个独立的开源项目，由任何公司独立维护。为了强调这一点并明确项目的治理结构，Prometheus 于 2016 年加入了 [云原生计算基金会](https://cncf.io/)，成为继 [Kubernetes](https://kubernetes.ac.cn/) 之后第二个托管项目。

Prometheus 将其指标作为时间序列数据收集和存储，即指标信息与记录时的时间戳以及可选的键值对（称为标签）一起存储。

### 2. 特性 {#2}

Prometheus 的主要特性包括：

*  
一个多维[数据模型](https://prometheus.ac.cn/docs/concepts/data_model/)，其中时间序列数据由指标名称和键/值对标识。  
*  
PromQL，一种灵活的[查询语言](https://prometheus.ac.cn/docs/prometheus/latest/querying/basics/)，用于利用这种多维性。  
*  
不依赖分布式存储；单个服务器节点是自主的。  
*  
时间序列收集通过 HTTP 上的拉取模型进行。  
*  
通过中间网关支持[推送时间序列](https://prometheus.ac.cn/docs/instrumenting/pushing/)。  
*  
通过服务发现或静态配置发现目标。  
*  
  支持多种图形和仪表盘模式。

### 3. 什么是指标（Metric）？ {#3-metric}

指标在通俗意义上是数值测量。术语"时间序列"是指随时间记录的变化。用户希望测量的内容因应用程序而异。对于 Web 服务器，可能是请求时间；对于数据库，可能是活动连接数或活动查询数等等。

指标在理解应用程序为何以某种方式工作方面发挥着重要作用。假设您正在运行一个 Web 应用程序并发现它运行缓慢。要了解应用程序发生了什么，您需要一些信息。例如，当请求数量很高时，应用程序可能会变慢。如果您拥有请求计数指标，则可以确定原因并增加服务器数量以处理负载。

### 4. 组件 {#4}

Prometheus 生态系统由多个组件组成，其中许多是可选的

*  
主 [Prometheus 服务器](https://github.com/prometheus/prometheus)，它抓取并存储时间序列数据。  
*  
用于对应用程序代码进行埋点的[客户端库](https://prometheus.ac.cn/docs/instrumenting/clientlibs/)。  
*  
一个用于支持短生命周期作业的[推送网关](https://github.com/prometheus/pushgateway)。  
*  
用于 HAProxy、StatsD、Graphite 等服务的专用[导出器](https://prometheus.ac.cn/docs/instrumenting/exporters/)。  
*  
一个用于处理告警的[告警管理器](https://github.com/prometheus/alertmanager)。  
*  
  各种支持工具。

大多数 Prometheus 组件都是用 [Go](https://golang.ac.cn/) 编写的，这使得它们易于构建并部署为静态二进制文件。

### 5. 架构 {#5}

此图展示了 Prometheus 及其部分生态系统组件的架构：

![image-20250729212709079.png](https://article-images.zsxq.com/FsirPjnx9er499VdFCCk08Z3mzAG)Prometheus 从已埋点的作业中抓取指标，可以直接抓取，也可以通过中间推送网关抓取短生命周期作业的指标。它将所有抓取的样本存储在本地，并根据这些数据运行规则，以聚合并记录现有数据中的新时间序列或生成告警。[Grafana](https://grafana.com/) 或其他 API 消费者可用于可视化收集到的数据。

### 6. 为什么选择拉而不是推？ {#6}

Prometheus 默认采用的是一种 **"拉模型（PullModel）"** 架构，它会**主动周期性地拉取被监控目标的指标数据** 。每个被监控的服务需要暴露一个支持 **Prometheus格式** 的 HTTP 接口，通常路径是 `/metrics`（如 Spring Boot 的 `/actuator/prometheus`）。

![iShot_2025-07-28_19.28.60.png](https://article-images.zsxq.com/FqcGwgbowDLgsnRv5RbeJSgu2SPC)

通过 HTTP 进行拉取有很多优点：

*  
根据需要启动监控实例，例如在本地开发时在笔记本电脑上启动一个 Prometheus 实例进行调试。  
*  
更容易、更可靠地判断某个目标是否宕机。  
*  
  可以手动访问目标服务的指标接口，直接通过浏览器检查其健康状态。

总体而言，Prometheus 认为拉模式略优于推模式，在考虑监控系统时，拉和推不应该成为主要考虑点。

对于必须推送的情况，Prometheus 同时也提供了 [Pushgateway](https://prometheus.io/docs/instrumenting/pushing/) 组件。

Docker 安装 Prometheus {#docker-prometheus}
-----------------------------------------

### 1. 简易版安装 {#1}

考虑到部分同学对 Docker 这些命令不太熟悉，常规安装 Prometheus 需要挂载配置文件地址，同时 Windows 和 Mac、Linux 路径方式还不太一样，这里Breathe希望在 **Docker启动命令中直接传入Prometheus配置内容** ，**不挂载本地配置文件** ，实现"**一条命令部署Prometheus并用自定义配置** "。

虽然 Prometheus 官方镜像不支持直接通过命令行参数传入完整配置，但是玩了个花活实现需求。使用 `echo` + `docker run` + `--entrypoint` 动态生成配置：

* 1.  
创建一个临时容器；  
* 2.  
用 `echo` 把配置写到容器内 `/etc/prometheus/prometheus.yml`；  
* 3.  
  然后启动 Prometheus。

Docker 命令如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker run -d \
      --name prometheus \
      -p 9090:9090 \
      --entrypoint sh \
      -e TZ=Asia/Shanghai \
      prom/prometheus:v2.51.1 \
      -c 'echo "
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    ​
    scrape_configs:
      - job_name: '\''prometheus'\''
        static_configs:
          - targets: ['\''localhost:9090'\'']
    ​
      - job_name: '\''onethread-app'\''
        metrics_path: '\''/actuator/prometheus'\''
        static_configs:
          - targets: ['\''host.docker.internal:18080'\'']
        scrape_interval: 10s
        scrape_timeout: 5s
    " > /etc/prometheus/prometheus.yml && /bin/prometheus --config.file=/etc/prometheus/prometheus.yml'
              
参数说明：

*  
`--entrypoint sh`：先让容器以 shell 启动；  
*  
`-c 'echo "... && /bin/prometheus ...'`：一口气完成写入配置并启动 Prometheus；  
*  
`'\''` 是 Bash 里的转义方法，用于生成单引号 `'`；  
*  
  整体是 shell 脚本拼接实现，不需要本地文件也能运行。

***如果想简易实现oneThread动态线程Metric采集，上面这条命令足够了。上述配置文件重写等同于：`yamlglobal:scrape_interval:15s#全局采集间隔evaluation_interval:15s#告警规则评估间隔​#采集任务配置scrape_configs:#Prometheus自身监控-job_name:'prometheus'static_configs:-targets:['localhost:9090']​#oneThread应用监控-job_name:'onethread-app'metrics_path:'/actuator/prometheus'static_configs:-targets:['host.docker.internal:18080','host.docker.internal:18081']#同一任务需要采集多IP，可以逗号分割scrape_interval:10sscrape_timeout:5s`因为Docker和宿主机的IP是不一样的，如果在Docker容器中运行Prometheus时，** 要访问宿主机的服务或端口* \*，推荐使用：`host.docker.internal`。在`Linux（Dockerv20.10+）`、`macOS`和`Windows`中，Docker提供了特殊的主机名，它可以在容器内解析为宿主机的IP。`host.docker.internal:18080`表示Prometheus容器去访问宿主机18080端口的服务。###2.运行检查浏览器访问<http://localhost:9090/targets>如果可以出现下述页面，即为运行成功。![image-20250729115229461.png](https://article-images.zsxq.com/FvHA-dDQaYKQ3S7w53wd79VC48QP)可以看到咱们在Prometheus配置文件中加的两个采集任务，都在列表上展示，并且State显示为`UP`健康状态。\>大家记得把onethread-nacos-cloud-example项目运行起来，要不然上面那个任务状态会显示失败。##Prometheus控制台操作指南###1.WebUI界面概览访问`http://localhost:9090`进入PrometheusWeb控制台，主要包含以下功能模块：\*\* Graph（图表查询）\*\*：

*  
**功能** ：执行 PromQL 查询，查看指标数据和图表。  
*  
  **用途** ：数据探索、问题排查、趋势分析。

**Alerts（告警管理）** ：

*  
**功能** ：查看当前告警状态和历史记录。  
*  
  **用途** ：告警监控、规则调试。

**Status（状态信息）** ：

*  
**Targets** ：查看采集目标状态。  
*  
**ServiceDiscovery** ：查看服务发现状态。  
*  
**Configuration** ：查看当前配置。  
*  
  **Rules** ：查看告警规则状态。

### 2. Graph 页面详细操作 {#2-graph}

**基础查询操作** ：

* 1.  
  **简单指标查询** ：  
  textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                      dynamic_thread_pool_active_size
                
显示所有线程池的活跃线程数。  
* 2.  
  **标签过滤查询** ：  
  textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                      dynamic_thread_pool_active_size{application_name="xxx"}
                
只显示特定应用的线程池数据。  
* 3.  
  **时间范围选择** ：
  *  
  使用页面顶部的时间选择器。  
  *  
  支持相对时间：5m、1h、1d、7d 等。  
  *  
    支持绝对时间：指定具体的开始和结束时间。

![image-20250729152434091.png](https://article-images.zsxq.com/Fjn8EPxMjy7oRv3zeR3jr8H7pMgR)

可以点击这个 [链接](http://localhost:9090/graph?g0.expr=dynamic_thread_pool_active_size&g0.tab=0&g0.display_mode=stacked&g0.show_exemplars=0&g0.range_input=30m) 跳转到查询该规则的页面，链接上带着 URL 参数，非常方便。

**图表显示选项** ：

*  
**Graph** ：时间序列图表，显示指标随时间的变化。  
*  
**Console** ：表格形式显示当前值。  
*  
  **Table** ：以表格形式显示所有时间序列。

### 3. Configuration 页面 {#3-configuration}

**查看当前配置** ：

Status → Configuration 页面显示当前生效的完整配置，包括：

*  
**全局配置** ：采集间隔、评估间隔等。  
*  
**采集任务配置** ：所有 scrape_configs 的详细配置。  
*  
  **告警规则配置** ：rule_files 中定义的规则。

**配置热重载** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 方法1：发送 HTTP 请求
    curl -X POST http://localhost:9090/-/reload
    ​
    # 方法2：发送系统信号
    docker exec prometheus kill -HUP 1
              
PromQL 查询语言实践 {#prom-ql}
------------------------

### 1. PromQL 基础语法 {#1-prom-ql}

**即时查询** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查询当前时刻的指标值
    dynamic_thread_pool_active_size
              
**范围查询** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查询过去5分钟的指标值
    dynamic_thread_pool_active_size[5m]
              
**标签匹配** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 精确匹配
    dynamic_thread_pool_active_size{job="onethread-app"}
    ​
    # 正则匹配
    dynamic_thread_pool_active_size{application_name=~".*example.*"}
    ​
    # 不等匹配
    dynamic_thread_pool_active_size{dynamic_thread_pool_id!="onethread-producer"}
              
### 2. oneThread 指标查询实例 {#2-one-thread}

**线程池活跃度监控** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查看所有线程池的活跃线程数
    dynamic_thread_pool_active_size
    ​
    # 查看特定应用的线程池活跃度
    dynamic_thread_pool_active_size{application_name="xxx"}
    ​
    # 计算线程池使用率
    dynamic_thread_pool_active_size / dynamic_thread_pool_maximum_size * 100
              
**队列状态监控** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查看队列当前长度
    dynamic_thread_pool_queue_size
    ​
    # 计算队列使用率
    dynamic_thread_pool_queue_size / dynamic_thread_pool_queue_capacity * 100
    ​
    # 查看队列剩余容量
    dynamic_thread_pool_queue_remaining_capacity
              
**任务执行统计** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 查看已完成任务数
    dynamic_thread_pool_completed_task_count
    ​
    # 计算任务完成速率（每秒）
    rate(dynamic_thread_pool_completed_task_count[5m])
    ​
    # 查看拒绝任务数
    dynamic_thread_pool_reject_count
              
### 3. 聚合函数应用 {#3}

**按应用聚合** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 计算每个应用的总活跃线程数
    sum by (application_name) (dynamic_thread_pool_active_size)
    ​
    # 计算每个应用的平均队列长度
    avg by (application_name) (dynamic_thread_pool_queue_size)
    ​
    # 查找每个应用中活跃线程数最多的线程池
    max by (application_name) (dynamic_thread_pool_active_size)
              
**时间窗口聚合** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 过去5分钟的平均活跃线程数
    avg_over_time(dynamic_thread_pool_active_size[5m])
    ​
    # 过去1小时的最大队列长度
    max_over_time(dynamic_thread_pool_queue_size[1h])
    ​
    # 过去10分钟的任务完成增长量
    increase(dynamic_thread_pool_completed_task_count[10m])
              
### 4. 复杂查询示例 {#4}

**线程池健康度评估** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 线程池压力指数（活跃线程数 + 队列长度）
    (dynamic_thread_pool_active_size + dynamic_thread_pool_queue_size) / dynamic_thread_pool_maximum_size
    ​
    # 识别高负载线程池（使用率超过80%）
    dynamic_thread_pool_active_size / dynamic_thread_pool_maximum_size > 0.8
    ​
    # 检测队列堆积严重的线程池
    dynamic_thread_pool_queue_size / dynamic_thread_pool_queue_capacity > 0.7
              
**多维度对比查询** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 对比不同线程池的性能表现
    topk(5, dynamic_thread_pool_active_size)
    ​
    # 查找最繁忙的应用
    topk(3, sum by (application_name) (dynamic_thread_pool_active_size))
    ​
    # 识别异常线程池（活跃线程数突然下降）
    dynamic_thread_pool_active_size < 
      avg_over_time(dynamic_thread_pool_active_size[1h]) * 0.5
              
### 5. PromQL 查询优化技巧 {#5-prom-ql}

**查询性能优化** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 避免：查询大时间范围的原始数据
    dynamic_thread_pool_active_size[24h]
    ​
    # 推荐：使用聚合函数减少数据量
    avg_over_time(dynamic_thread_pool_active_size[24h])
              
**标签选择优化** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 避免：使用正则匹配大量标签
    {__name__=~"dynamic_thread_pool_.*"}
    ​
    # 推荐：使用精确的标签匹配
    dynamic_thread_pool_active_size{application_name="xxx"}
              
文末总结
----

通过 Prometheus 监控系统，oneThread 框架真正实现了"可观测性"的目标。运维团队不仅能够实时了解线程池的运行状态，还能通过历史数据分析发现潜在问题，为系统优化提供数据支撑。结合前面介绍的本地日志监控和 Micrometer 指标监控，oneThread 框架已经构建了一套完整的监控体系。无论是开发调试、生产运维，还是性能分析，都能找到合适的监控工具和方法。

至此，oneThread 动态线程池框架的 Prometheus 监控集成就全部介绍完了，下一章节会带着大家一起部署 Grafana，将 Prometheus 时序数据展示在 Grafana 大屏。

完结，撒花 🎉

