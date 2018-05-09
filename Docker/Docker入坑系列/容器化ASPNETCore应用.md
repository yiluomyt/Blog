# 容器化ASP.NET Core应用

本文参考了[示例](https://docs.docker.com/engine/examples/dotnetcore/)。

## 最简单的Web API应用

.net core作为微软新一代的跨平台开源环境，无论是官方还是社区都对其的容器化有极大的热情。

首先，我们利用dotnet CLI工具创建一个新的Web API应用，并在本地测试一下。

```PowerShell
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

![docker-3-0](../../Images/Docker/Docker入坑系列/容器化ASPNETCore应用/api样例.png)

显然，和之前的python应用不同，.net core需要经过build才能运行。为了方便build，微软官方也推出了一个build用的镜像`microsoft/aspnetcore-build`。

这里，参考示例，我们利用`microsoft/aspnetcore-build`来build应用，并将输出复制到`microsoft/aspnetcore`中来运行。

```Dockerfile
# build镜像
FROM microsoft/aspnetcore-build AS build-env
WORKDIR /app

# 还原Nuget包
COPY *.csproj ./
RUN dotnet restore

# 复制所有文件
COPY . ./
# 发布应用
RUN dotnet publish -c Release -o out

# 运行镜像
FROM microsoft/aspnetcore
WORKDIR /app
# 从build镜像中复制输出
COPY --from=build-env /app/out .
# 设置入口
ENTRYPOINT [ "dotnet", "aspnetapp.dll" ]
```

然后，就是`docker build`了？不，在这里我们还要多加一步，因为之前我们已经在本地运行过了，.net core会在当前目录下创建**/bin**和**/obj**来储存二进制文件和中间文件，而这部分在我们构建镜像时是不需要的，我们可以通过**.dockerignore**来忽略他们（就和**.gitignore**一样）。

在当前目录下，创建一个**.dockerignore**文件，键入以下内容。

```txt
/bin
/obj
```

最后，就是我们熟悉的`docker build -t demo:aspnetapp .`。但注意在build结束后，你会发现除了我们指定生成的demo:aspnetapp镜像外还有一个`<none>:<none>`的dangling镜像，这个镜像就是我们在**Dockerfile**前半部分中用的build环境，如果觉得不再需要了，可以通过`docker image prune`来删除所有dangling镜像。

最后的最后，注意一个细节。。也不清楚microsoft/aspnetcore中的环境是怎么配置的，按照以上步骤生成的镜像在运行时监听的是80端口。所以，在启动实例时，记得用`-p 80:80`来映射80端口。