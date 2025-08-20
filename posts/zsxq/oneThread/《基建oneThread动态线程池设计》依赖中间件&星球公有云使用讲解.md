2025年06月25日 20:57  
依赖中间件\&星球公有云使用讲解，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★☆☆☆☆ - 简单  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：因为受本地环境等多因素影响，安装众多中间件无疑会影响大家学习进度，所以星球提供了无本地依赖的中间件云服务版本。

课程目录如下所示：

*  
公有云中间件  
*  
  本地中间件安装

公有云中间件
------

### 1. Nacos {#1-nacos}

> 公有云中间件 Nacos 如何在项目中使用，详情查看后端快速启动文章。

*  
🌟IDEA VM 参数配置连接地址：common-nacos-dev.magestack.cn:8848  
*  
如果想要在浏览器访问：<http://common-nacos-dev.magestack.cn:8848/nacos/index.html>  
*  
  首次登录可能需要用户名密码：nacos/nacos

![image-20250625142127582.png](https://article-images.zsxq.com/Fv19-9nJ5zMdaVt2jV5NQ6nZ9Vg4 "image-20250625142127582.png")

本地中间件安装
-------

### 1. 安装说明 {#1}

对于没有安装过上述中间件的同学，不建议大家在本地安装，可能会有各式各样的问题。建议先使用星球通用云中间件，首先让应用正常运行，然后再进行学习。
> 本地中间件安装使用 Docker 方式。

### 2. Nacos {#2-nacos}

Windows、Linux 以及 Mac M1 电脑均通过以下 Docker 命令启动 MySQL 实例，Docker 会自动选择匹配架构的镜像（官方镜像已支持多平台）：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker run -d \
      --name nacos \
      -p 8848:8848 \
      -p 9848:9848 \
      -e TIME_ZONE='Asia/Shanghai' \
      -e MODE=standalone \
      -e JVM_XMS=128m \
      -e JVM_XMX=256m \
      nacos/nacos-server:v2.1.0
              
Docker 启动配置参数说明：

*  
`-d` **后台运行模式** ：以守护进程方式运行容器，不占用当前终端窗口。  
*  
`--name nacos` **容器名称标识** ：指定容器名称为 `nacos`，便于后续管理操作。  
*  
`-p 8848:8848` **关键端口映射** ：将宿主机的 **8848端口** 映射到 Nacos 容器的 HTTP 接口和控制台端口。  
*  
`-p 9848:9848` **gRPC通信端口** ：Nacos 2.0+ 必需端口（用于客户端与服务器通信），不开放会导致服务不可用。  
*  
`-e TIME_ZONE='Asia/Shanghai'` **时区** ：强制容器使用 **中国时区（GMT+8）** 。  
*  
`-e MODE=standalone` **运行模式配置** ：设置环境变量启动 **单机模式** （集群模式需改为 `cluster`）。  
*  
`-e JVM_XMS=128m` **初始堆内存分配** ：Nacos 启动时立即占用的内存。  
*  
`-e JVM_XMX=128m` **最大堆内存上限** ：允许 Nacos JVM 使用的最大内存。  
*  
  `nacos/nacos-server:2.1.0` **镜像定义** ：指定使用的官方 Nacos 镜像及版本号。

上面的 `XMS` 和 `XMX` 在本地电脑跑的时候，可以不加限制。如果在服务器上运行，在内存不富裕的情况下还是要限制内存使用。

完结，撒花 🎉  
依赖中间件\&星球公有云使用讲解，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★☆☆☆☆ - 简单  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：因为受本地环境等多因素影响，安装众多中间件无疑会影响大家学习进度，所以星球提供了无本地依赖的中间件云服务版本。

课程目录如下所示：

*  
公有云中间件  
*  
  本地中间件安装

公有云中间件
------

### 1. Nacos {#1-nacos}

> 公有云中间件 Nacos 如何在项目中使用，详情查看后端快速启动文章。

*  
🌟IDEA VM 参数配置连接地址：common-nacos-dev.magestack.cn:8848  
*  
如果想要在浏览器访问：<http://common-nacos-dev.magestack.cn:8848/nacos/index.html>  
*  
  首次登录可能需要用户名密码：nacos/nacos

![image-20250625142127582.png](https://article-images.zsxq.com/Fv19-9nJ5zMdaVt2jV5NQ6nZ9Vg4 "image-20250625142127582.png")

本地中间件安装
-------

### 1. 安装说明 {#1}

对于没有安装过上述中间件的同学，不建议大家在本地安装，可能会有各式各样的问题。建议先使用星球通用云中间件，首先让应用正常运行，然后再进行学习。
> 本地中间件安装使用 Docker 方式。

### 2. Nacos {#2-nacos}

Windows、Linux 以及 Mac M1 电脑均通过以下 Docker 命令启动 MySQL 实例，Docker 会自动选择匹配架构的镜像（官方镜像已支持多平台）：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    docker run -d \
      --name nacos \
      -p 8848:8848 \
      -p 9848:9848 \
      -e TIME_ZONE='Asia/Shanghai' \
      -e MODE=standalone \
      -e JVM_XMS=128m \
      -e JVM_XMX=256m \
      nacos/nacos-server:v2.1.0
              
Docker 启动配置参数说明：

*  
`-d` **后台运行模式** ：以守护进程方式运行容器，不占用当前终端窗口。  
*  
`--name nacos` **容器名称标识** ：指定容器名称为 `nacos`，便于后续管理操作。  
*  
`-p 8848:8848` **关键端口映射** ：将宿主机的 **8848端口** 映射到 Nacos 容器的 HTTP 接口和控制台端口。  
*  
`-p 9848:9848` **gRPC通信端口** ：Nacos 2.0+ 必需端口（用于客户端与服务器通信），不开放会导致服务不可用。  
*  
`-e TIME_ZONE='Asia/Shanghai'` **时区** ：强制容器使用 **中国时区（GMT+8）** 。  
*  
`-e MODE=standalone` **运行模式配置** ：设置环境变量启动 **单机模式** （集群模式需改为 `cluster`）。  
*  
`-e JVM_XMS=128m` **初始堆内存分配** ：Nacos 启动时立即占用的内存。  
*  
`-e JVM_XMX=128m` **最大堆内存上限** ：允许 Nacos JVM 使用的最大内存。  
*  
  `nacos/nacos-server:2.1.0` **镜像定义** ：指定使用的官方 Nacos 镜像及版本号。

上面的 `XMS` 和 `XMX` 在本地电脑跑的时候，可以不加限制。如果在服务器上运行，在内存不富裕的情况下还是要限制内存使用。

完结，撒花 🎉

