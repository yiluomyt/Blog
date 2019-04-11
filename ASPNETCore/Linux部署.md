---
tags:
  - ASP.NET Core
  - Linux
date: 2018-05-09
---

# Linux 部署

## 腾讯云

如果只是玩玩的话，可以考虑腾讯云按量计费的服务器，0.23 元/小时，还是很划算的。

若有需要轻度的生产环境的话，可以注册一下学生认证，1 核 1G 现在 10 元/月。[云+校园计划](https://cloud.tencent.com/act/campus)

如果非常在意性价比的话，阿里云貌似是 10 元/月的 1 核 2G。

我所选的配置

![CVM配置](../Images/ASPNETCore/Linux部署/CVM配置.png)

## 安全组

安全组其实就是云主机的防火墙，我们需要新建一个安全组，打开 22 端口(用于连接 CVM)和 5000 端口(WEB 应用绑定的端口)，这样我们才能从外网访问的到该服务器。

简单的步骤如下，

1. 新建安全组

   ![新建安全组](../Images/ASPNETCore/Linux部署/新建安全组.png)

2. 添加入站规则

   ![添加入站规则](../Images/ASPNETCore/Linux部署/添加入站规则.png)

3. 关联实例

   将云主机关联到安全组，这就不截图了吧。

## 安装环境

在创建好之后，我们可以用 Putty 远程连接上 CVM。(这里注意在 Putty 中用中文输入法可能会卡死)

首先，用`yum update -y`更新所有已安装的软件，以防一些软件版本太老导致一些诡异的问题。

在更新完之后，可以参照微软的[教程](https://www.microsoft.com/net/learn/get-started/linux/centos)安装.net core 环境。

> 在安装时，可能会出现短暂的假死。

至此，.net core 环境已安装完毕，可以通过`dotnet --info`查看相关信息。

![dotnet-info](../Images/ASPNETCore/Linux部署/dotnet-info.png)

## 发布 WEB 应用

这里我们就用之前写的[简易版 PFSign](https://github.com/panfengstudio/workshop/tree/2018/05/05)作为示例。

Clone 后切到根目录，然后`dotnet publish -c Release`指定以生产环境发布。

很简单是吗？那就等着报错吧。。
有兴趣的可以试试，若这样直接上去会在服务器上得到如下提示

![默认不支持IPv6](../Images/ASPNETCore/Linux部署/默认不支持IPv6.png)

在提示中可以看到，'http://localhost:5000' 不能绑定到 IPv6。

> 这是因为 localhost 代表的是 127.0.0.1 是一个 IPv4 地址，自然不能绑定到 IPv6 的地址上。

解决这个问题也很简单，我们只需在'Program.cs'中添加上以下一行即可。

```csharp
public static IWebHost BuildWebHost(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .UseStartup<Startup>()
        // 添加该行
        // 默认配置为http://localhost:5000
        // 这里的*代表自动选择指向本地的IP地址
        .UseUrls("http://*:5000")
        .Build();
```

## 上传 WEB 应用

这里我们可以利用 Xftp 通过 SFTP 协议上传文件。

之前找到之前发布的 WEB 应用`\bin\Release\netcoreapp2.0\publish`，直接右键上传整个文件夹。

## 启动 WEB 应用

回到 Putty 的命令行界面，用 cd 指令切到 publish 文件夹的目录中，运行`dotnet workshop.dll`，这里的 workshop.dll 是之前发布的应用入口文件。

![启动WEB应用](../Images/ASPNETCore/Linux部署/启动WEB应用.png)

至此，应该简单的部署流程就结束了。

我们已经可以用 http://(公网ip):5000 来访问我们的应用。
