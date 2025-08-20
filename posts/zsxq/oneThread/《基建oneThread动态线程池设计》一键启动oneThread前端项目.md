2025年06月28日 21:29  
一键启动oneThread前端项目，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节指导大家 Windows 电脑如何通过 Nginx 快速启动前端项目，以及 Windows 和 Mac 电脑如何通过源码安装方式启动。

课程目录如下所示：

*  
快速启动  
*  
安装 Nodejs  
*  
  启动前端项目

在启动前端项目之前，请先确保后端服务已运行。如果后端尚未启动，请先完成后端项目的启动流程。

快速启动
----

如果你是 Windows 电脑，可以通过 Nginx 快速启动的方式部署 oneThread 前端工程。下载这个 Nginx 压缩包，并解压。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    通过网盘分享的文件：nageoffer-nginx-1.3.0.1.zip
    链接: https://pan.baidu.com/s/1hEbVSRKE_FsOT_PyI1eLgQ?pwd=6688 提取码: 6688

然后双击 nginx 图标执行启动逻辑。

![image-20250628211040288.png](https://article-images.zsxq.com/FpCWI_vvBzibbaughyv-LFb3VsRP "image-20250628211040288.png")

浏览器访问 `localhost` 如果返回 nginx 页面，则证明 nginx 启动成功。

![image-20250628211106129.png](https://article-images.zsxq.com/FrnPRax1efZDF15G08SAV6h8553B "image-20250628211106129.png")

浏览器访问 `http://localhost:5176/` 跳转 oneThread 登录页面，既为前端启动成功，可以正常使用系统。

**如果想自己本地安装nodejs的方式请参考下文。**

**安装Nodejs** {#nodejs}
----------------------

oneThread 项目前端安装 `20.15.0` 版本 Nodejs\*\*，\*\* 其它版本存在版本不兼容等问题。

### 1. 电脑中没安装过 Nodejs {#1-nodejs}

打开下述链接 🔗 访问 Nodejs 下载页面：<https://nodejs.org/download/release/v20.15.0>

根据不同的操作系统下载安装不同的包。下载完成后，双击打开对应的安装软件包，应该就可以通过鼠标点点点的方式进行安装。

*  
Windows 电脑：node-v20.15.0-win-x86.zip  
*  
  Mac 电脑：node-v20.15.0.pkg

安装完成后，终端输入 `node -v` 检查是否安装成功，输入命令返回以下信息表示安装成功。

![image-20250628205857509.png](https://article-images.zsxq.com/FrjUFHiCKGW5s6EGvzvajU1Zgtxa "image-20250628205857509.png")

### 2. 电脑中已有 Nodejs {#2-nodejs}

如果电脑中已经存在 Nodejs 环境，检查版本是否在 20.15.0 版本，如果不是需要安装 Nodejs 多版本控制组件，下载多个 Nodejs 共存，并通过命令切换。或者简单点，直接删除之前版本，下载 20.15.0 版本即可。

#### 2.1 Mac 电脑教程 {#2-1-mac}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 安装 n 组件
    npm install -g n
    ​
    # 安装完成后，查看 n -V
    n -V

注意，-V 是大写的，出现以下信息代表安装成功。

![image-20250628210239596.png](https://article-images.zsxq.com/Flt255Chx9_1toH2vqLjM0FgPbRd "image-20250628210239596.png")

下载多版本 Nodejs 组件。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 安装 20.15.0 版本的 Nodejs
    n 20.15.0
    ​
    # 安装完成后，执行切换
    sudo n 20.15.0
    ​
    # 切换成功后，输入 node -v 查看版本是否正确
    node -v

#### 2.2 Windows 电脑教程 {#2-2-windows}

点击查看 [Windows 电脑切换 Nodejs 多版本参考链接](https://blog.csdn.net/qq_38405436/article/details/132279098)。

启动前端项目
------

下载 [onethread-dashboard](https://gitee.com/nageoffer/onethread-dashboard) 前端项目，依次执行下述命令。

### 1. 通过 npm 安装依赖 {#1-npm}

进入 onethread-dashboard 项目的根目录。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 使用项目指定的pnpm版本进行依赖安装
    npm i -g corepack
    ​
    # 安装依赖
    pnpm install

### 2. 启动项目 {#2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 启动项目
    pnpm dev

### 3. 访问前端页面 {#3}

通过访问 [前端界面](http://localhost:5777) 进行登录体验功能。

完结，撒花 🎉  
一键启动oneThread前端项目，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节指导大家 Windows 电脑如何通过 Nginx 快速启动前端项目，以及 Windows 和 Mac 电脑如何通过源码安装方式启动。

课程目录如下所示：

*  
快速启动  
*  
安装 Nodejs  
*  
  启动前端项目

在启动前端项目之前，请先确保后端服务已运行。如果后端尚未启动，请先完成后端项目的启动流程。

快速启动
----

如果你是 Windows 电脑，可以通过 Nginx 快速启动的方式部署 oneThread 前端工程。下载这个 Nginx 压缩包，并解压。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    通过网盘分享的文件：nageoffer-nginx-1.3.0.1.zip
    链接: https://pan.baidu.com/s/1hEbVSRKE_FsOT_PyI1eLgQ?pwd=6688 提取码: 6688

然后双击 nginx 图标执行启动逻辑。

![image-20250628211040288.png](https://article-images.zsxq.com/FpCWI_vvBzibbaughyv-LFb3VsRP "image-20250628211040288.png")

浏览器访问 `localhost` 如果返回 nginx 页面，则证明 nginx 启动成功。

![image-20250628211106129.png](https://article-images.zsxq.com/FrnPRax1efZDF15G08SAV6h8553B "image-20250628211106129.png")

浏览器访问 `http://localhost:5176/` 跳转 oneThread 登录页面，既为前端启动成功，可以正常使用系统。

**如果想自己本地安装nodejs的方式请参考下文。**

**安装Nodejs** {#nodejs}
----------------------

oneThread 项目前端安装 `20.15.0` 版本 Nodejs\*\*，\*\* 其它版本存在版本不兼容等问题。

### 1. 电脑中没安装过 Nodejs {#1-nodejs}

打开下述链接 🔗 访问 Nodejs 下载页面：<https://nodejs.org/download/release/v20.15.0>

根据不同的操作系统下载安装不同的包。下载完成后，双击打开对应的安装软件包，应该就可以通过鼠标点点点的方式进行安装。

*  
Windows 电脑：node-v20.15.0-win-x86.zip  
*  
  Mac 电脑：node-v20.15.0.pkg

安装完成后，终端输入 `node -v` 检查是否安装成功，输入命令返回以下信息表示安装成功。

![image-20250628205857509.png](https://article-images.zsxq.com/FrjUFHiCKGW5s6EGvzvajU1Zgtxa "image-20250628205857509.png")

### 2. 电脑中已有 Nodejs {#2-nodejs}

如果电脑中已经存在 Nodejs 环境，检查版本是否在 20.15.0 版本，如果不是需要安装 Nodejs 多版本控制组件，下载多个 Nodejs 共存，并通过命令切换。或者简单点，直接删除之前版本，下载 20.15.0 版本即可。

#### 2.1 Mac 电脑教程 {#2-1-mac}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 安装 n 组件
    npm install -g n
    ​
    # 安装完成后，查看 n -V
    n -V

注意，-V 是大写的，出现以下信息代表安装成功。

![image-20250628210239596.png](https://article-images.zsxq.com/Flt255Chx9_1toH2vqLjM0FgPbRd "image-20250628210239596.png")

下载多版本 Nodejs 组件。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 安装 20.15.0 版本的 Nodejs
    n 20.15.0
    ​
    # 安装完成后，执行切换
    sudo n 20.15.0
    ​
    # 切换成功后，输入 node -v 查看版本是否正确
    node -v

#### 2.2 Windows 电脑教程 {#2-2-windows}

点击查看 [Windows 电脑切换 Nodejs 多版本参考链接](https://blog.csdn.net/qq_38405436/article/details/132279098)。

启动前端项目
------

下载 [onethread-dashboard](https://gitee.com/nageoffer/onethread-dashboard) 前端项目，依次执行下述命令。

### 1. 通过 npm 安装依赖 {#1-npm}

进入 onethread-dashboard 项目的根目录。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 使用项目指定的pnpm版本进行依赖安装
    npm i -g corepack
    ​
    # 安装依赖
    pnpm install

### 2. 启动项目 {#2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    # 启动项目
    pnpm dev

### 3. 访问前端页面 {#3}

通过访问 [前端界面](http://localhost:5777) 进行登录体验功能。

完结，撒花 🎉

