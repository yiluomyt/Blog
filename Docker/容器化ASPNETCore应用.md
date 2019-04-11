---
tags:
  - Docker
  - ASP.NET Core
date: 2018-11-16
---

# 容器化 ASP.NET Core 应用

> 2018/11/16 更新 ASP.NET Core 2.1 的容器化

## 最简单的 Web API 应用

.net core 作为微软新一代的跨平台开源环境，无论是官方还是社区都对其的容器化有极大的热情。

首先，我们利用 dotnet CLI 工具创建一个新的 Web API 应用，并在本地测试一下。

```powershell
# 新建项目
dotnet new webapi -o aspnetapp
# 切换目录
cd aspnetapp
# 构建
dotnet build
# 运行
dotnet run
```

打开浏览器键入[http://localhost:5000/api/values](http://localhost:5000/api/values)，我们可以看到返回的样例。

![docker-3-0](../../Images/Docker/容器化ASPNETCore应用/api样例.png)

显然，和之前的 python 应用不同，.net core 需要经过 build 才能运行。因此，我们可以先拉取完整的 SDK 镜像`microsoft/dotnet:2.1-sdk`来进行编译，以免因为在不同环境下 build 产生的问题。而在实际运行过程中，考虑到镜像容量以及运行时优化，我们应该采用专门的 runtime 镜像`microsoft/dotnet:2.1-aspnetcore-runtime`。

根据以上两个要求，我们可以给出以下 Dockerfile（其实基本就是 VS 自动生成的）

> 因为一些原因需要使用`DateTime.Now`，这里将最终镜像的时区设置为了 Asia/Shanghai，若不需要，将倒数 2/3 行删去即可。

```Dockerfile
# 拉取runtime镜像
FROM microsoft/dotnet:2.1-aspnetcore-runtime AS base
# 限定工作目录
WORKDIR /app
# 开放80端口
EXPOSE 80

# 拉取SDK镜像
FROM microsoft/dotnet:2.1-sdk AS build
WORKDIR /src
# 复制项目文件
COPY ["aspnetapp.csproj", "."]
# 还原项目(restore nuget package)
RUN dotnet restore
# 将剩余代码复制到容器中
COPY . .
# 以Release模式编译项目
RUN dotnet build -c Release -o /app

# 发布项目
FROM build AS publish
RUN dotnet publish -c Release -o /app

FROM base AS final
WORKDIR /app
# 复制发布后的文件
COPY --from=publish /app .
# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
# 设置容器入口
ENTRYPOINT ["dotnet", "aspnetapp.dll"]
```

然后，就是`docker build`了？不，在这里我们还要多加一步，因为之前我们已经在本地运行过了，.net core 会在当前目录下创建/bin 和/obj 来储存二进制文件和中间文件，而这部分在我们构建镜像时是不需要的，我们可以通过`.dockerignore`来忽略他们（就和.gitignore 一样）。

在当前目录下，创建一个`.dockerignore`文件，键入以下内容。

```
/bin
/obj
```

最后，就是我们熟悉的`docker build -t demo:aspnetapp .`。但注意在 build 结束后，你会发现除了我们指定生成的 demo:aspnetapp 镜像外还有一个`<none>:<none>`的 dangling 镜像，这个镜像就是我们在**Dockerfile**前半部分中用的 build 环境，如果觉得不再需要了，可以通过`docker image prune`来删除所有 dangling 镜像。

最后的最后，注意一个细节。。即使在 Dockerfile 中 EXPOSE 了 80 端口，在`docker run`的命令中，还是需要通过`-p 80:80`来映射到宿主机。
