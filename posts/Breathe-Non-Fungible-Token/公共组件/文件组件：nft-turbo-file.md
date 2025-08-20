![[Pasted image 20250819155651.png]]

我们的项目需要在用户模块管理的时候进行头像修改，我们把上传文件相关的内容封装到文件上传组件中。主要是封装了阿里云的oss

oss配置信息放在nacos上，通过spring 自动注入

然后构造成一个ossService的bean

