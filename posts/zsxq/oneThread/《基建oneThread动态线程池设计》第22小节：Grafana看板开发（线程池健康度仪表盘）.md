2025年08月03日 20:38  
Grafana看板开发（线程池健康度仪表盘），元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

课程目录如下所示：

*  
前言  
*  
Docker 安装 Grafana  
*  
Grafana 导入控制台  
*  
onethread-dashboard-dev Grafana 地址替换  
*  
  文末总结

前言
---

在日常监控场景中，**Grafana + Prometheus** 是一套常见的可观测性组合。通过几条简单的命令，演示如何使用 Docker 快速启动 Grafana 并与 Prometheus 容器建立连接。
> 需要确保大家已经看过上一章节，并且名为 prometheus 的 Docker 容器正在运行中。

Docker 安装 Grafana {#docker-grafana}
-----------------------------------

### 1. 创建 Docker 网络 {#1-docker}

为了让 Grafana 能够通过容器名访问 Prometheus，我们先创建一个共享网络 `monitoring`：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker network create monitoring
              
该网络将作为 Prometheus 与 Grafana 的通信桥梁。

### 2. Prometheus 加入网络 {#2-prometheus}

将已运行的 `prometheus` 容器连接到 `monitoring` 网络：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker network connect monitoring prometheus
              
### 3. 创建 Grafana {#3-grafana}

运行以下命令，启动指定版本的 Grafana 并加入同一网络：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker run -d \
      --name grafana \
      --network monitoring \
      -p 3000:3000 \
      grafana/grafana:9.0.5
              
访问 <http://127.0.0.1:3000> 登录 Grafana 控制台，用户名和密码分别是 admin/admin。
> 首次登录后会提示修改用户名和密码，Skip 跳过即可。

Grafana 导入控制台 {#grafana}
------------------------

### 1. 创建 Prometheus 数据源 {#1-prometheus}

Grafana 支持几十种数据源，这里咱们选择 Prometheus 数据源进行创建。

![image-20250803164449413.png](https://article-images.zsxq.com/Fvt6bJc_28LtCc5d6z6qT3riKoMW)

HTTP URL 处填写 <http://prometheus:9090> 即可，通过 Docker 内部网络进行通信，会自动将 prometheus 解析为对应的 IP 地址。

![image-20250803164749776.png](https://article-images.zsxq.com/FthwNZeWFB1XtLJOQq39SZx6MJqz)

划到最下面，点击 `Save % test` 按钮，出现上述绿色弹框，即可创建成功。

![image-20250803164927027.png](https://article-images.zsxq.com/FuWeshGQcZzxpj7ln9pT9w6Dkbxa)

### 2. 导入 DashBoard 模板 {#2-dash-board}

点击 Dashboard-Browse 按钮进入控制台页面，并点击 Import 按钮进行导入操作。

![image-20250803181645403.png](https://article-images.zsxq.com/FhvspSMJtjmrABP6gEKoaOU7FdFV)

通过下述百度网盘分享的文件下载 oneThread Grafana 的大屏 JSON 文件。
> 通过网盘分享的文件：onethread 链接: <https://pan.baidu.com/s/1fEWATeXAercyBcse09Kjgw?pwd=6688> 提取码: 6688

上传 JSON 文件并选择 Prometheus 数据源。

![image-20250803182713550.png](https://article-images.zsxq.com/FmYdMEva7Zb5znVVTlDi_wUqQkUh)

如果你的 onethread Nacos 项目一直在产出指标数据，并且 Prometheus 也在进行采集，那么导入后就会得到类似于下述的监控大盘。

![image-20250803182826936.png](https://article-images.zsxq.com/FiQL0DKg2pvTnTWJkV0OLW7JTusc)

如果存在监控多个项目和多个线程池，上面的 `application_name` 和 `dynamic_thread_pool_id` 可以进行选择。如果单个应用起了多个实例（集群部署），这个监控图表会有类似于下面这种效果，IP 展示会变成多个。

![image-20250803184224515.png](https://article-images.zsxq.com/FleycLjax_xwW261gIf4sm2B6zD6)

onethread-dashboard-dev Grafana 地址替换 {#onethread-dashboard-dev-grafana}
-----------------------------------------------------------------------

在 dashboard-dev 项目的配置文件中，有这么一个配置：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      grafana:
        # 如果本地有安装 Grafana 展示，可以替换为本地路径
        url: http://grafana.nageoffer.com/d/gxBvKxYNz/7adffa3?orgId=1&from=now-6h&to=now&timezone=browser&var-application_name=nacos-cloud-example&var-dynamic_thread_pool_id=onethread-consumer&refresh=5s&theme=light&kiosk=true
              
如果你打算将项目部署到公网环境，可以将上述 URL 替换为你自己部署的 Grafana 地址。前端展示的线程池监控页面，就是通过接口动态获取该参数来完成跳转与展示的。

文末总结
----

本文介绍了通过 Docker 快速部署 Grafana，并完成与 Prometheus 的打通与可视化配置。同时，我们还讲解了如何导入 oneThread 的官方 Dashboard 模板，并将其地址嵌入到项目配置中，实现线程池监控视图的一键接入。

一旦配置完成，无论是线程池的活跃线程数、任务堆积情况，还是拒绝策略告警，皆可在可视化界面中一目了然，帮助开发者第一时间感知系统运行状况。

完结，撒花 🎉  
Grafana看板开发（线程池健康度仪表盘），元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

课程目录如下所示：

*  
前言  
*  
Docker 安装 Grafana  
*  
Grafana 导入控制台  
*  
onethread-dashboard-dev Grafana 地址替换  
*  
  文末总结

前言
---

在日常监控场景中，**Grafana + Prometheus** 是一套常见的可观测性组合。通过几条简单的命令，演示如何使用 Docker 快速启动 Grafana 并与 Prometheus 容器建立连接。
> 需要确保大家已经看过上一章节，并且名为 prometheus 的 Docker 容器正在运行中。

Docker 安装 Grafana {#docker-grafana}
-----------------------------------

### 1. 创建 Docker 网络 {#1-docker}

为了让 Grafana 能够通过容器名访问 Prometheus，我们先创建一个共享网络 `monitoring`：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker network create monitoring
              
该网络将作为 Prometheus 与 Grafana 的通信桥梁。

### 2. Prometheus 加入网络 {#2-prometheus}

将已运行的 `prometheus` 容器连接到 `monitoring` 网络：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker network connect monitoring prometheus
              
### 3. 创建 Grafana {#3-grafana}

运行以下命令，启动指定版本的 Grafana 并加入同一网络：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker run -d \
      --name grafana \
      --network monitoring \
      -p 3000:3000 \
      grafana/grafana:9.0.5
              
访问 <http://127.0.0.1:3000> 登录 Grafana 控制台，用户名和密码分别是 admin/admin。
> 首次登录后会提示修改用户名和密码，Skip 跳过即可。

Grafana 导入控制台 {#grafana}
------------------------

### 1. 创建 Prometheus 数据源 {#1-prometheus}

Grafana 支持几十种数据源，这里咱们选择 Prometheus 数据源进行创建。

![image-20250803164449413.png](https://article-images.zsxq.com/Fvt6bJc_28LtCc5d6z6qT3riKoMW)

HTTP URL 处填写 <http://prometheus:9090> 即可，通过 Docker 内部网络进行通信，会自动将 prometheus 解析为对应的 IP 地址。

![image-20250803164749776.png](https://article-images.zsxq.com/FthwNZeWFB1XtLJOQq39SZx6MJqz)

划到最下面，点击 `Save % test` 按钮，出现上述绿色弹框，即可创建成功。

![image-20250803164927027.png](https://article-images.zsxq.com/FuWeshGQcZzxpj7ln9pT9w6Dkbxa)

### 2. 导入 DashBoard 模板 {#2-dash-board}

点击 Dashboard-Browse 按钮进入控制台页面，并点击 Import 按钮进行导入操作。

![image-20250803181645403.png](https://article-images.zsxq.com/FhvspSMJtjmrABP6gEKoaOU7FdFV)

通过下述百度网盘分享的文件下载 oneThread Grafana 的大屏 JSON 文件。
> 通过网盘分享的文件：onethread 链接: <https://pan.baidu.com/s/1fEWATeXAercyBcse09Kjgw?pwd=6688> 提取码: 6688

上传 JSON 文件并选择 Prometheus 数据源。

![image-20250803182713550.png](https://article-images.zsxq.com/FmYdMEva7Zb5znVVTlDi_wUqQkUh)

如果你的 onethread Nacos 项目一直在产出指标数据，并且 Prometheus 也在进行采集，那么导入后就会得到类似于下述的监控大盘。

![image-20250803182826936.png](https://article-images.zsxq.com/FiQL0DKg2pvTnTWJkV0OLW7JTusc)

如果存在监控多个项目和多个线程池，上面的 `application_name` 和 `dynamic_thread_pool_id` 可以进行选择。如果单个应用起了多个实例（集群部署），这个监控图表会有类似于下面这种效果，IP 展示会变成多个。

![image-20250803184224515.png](https://article-images.zsxq.com/FleycLjax_xwW261gIf4sm2B6zD6)

onethread-dashboard-dev Grafana 地址替换 {#onethread-dashboard-dev-grafana}
-----------------------------------------------------------------------

在 dashboard-dev 项目的配置文件中，有这么一个配置：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      grafana:
        # 如果本地有安装 Grafana 展示，可以替换为本地路径
        url: http://grafana.nageoffer.com/d/gxBvKxYNz/7adffa3?orgId=1&from=now-6h&to=now&timezone=browser&var-application_name=nacos-cloud-example&var-dynamic_thread_pool_id=onethread-consumer&refresh=5s&theme=light&kiosk=true
              
如果你打算将项目部署到公网环境，可以将上述 URL 替换为你自己部署的 Grafana 地址。前端展示的线程池监控页面，就是通过接口动态获取该参数来完成跳转与展示的。

文末总结
----

本文介绍了通过 Docker 快速部署 Grafana，并完成与 Prometheus 的打通与可视化配置。同时，我们还讲解了如何导入 oneThread 的官方 Dashboard 模板，并将其地址嵌入到项目配置中，实现线程池监控视图的一键接入。

一旦配置完成，无论是线程池的活跃线程数、任务堆积情况，还是拒绝策略告警，皆可在可视化界面中一目了然，帮助开发者第一时间感知系统运行状况。

完结，撒花 🎉

