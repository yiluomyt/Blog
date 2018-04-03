# Docker入坑系列-1

## 启动MySQL实例

对我个人来说，使用Docker的最大需求就是可以简化一些应用的安装，特别是数据库之类的－_－b。所以，使用Docker的第一个Demo就是启动一个MySQL实例。

```powershell
# 拉取MySQL镜像
docker pull mysql
# 启动MySQL实例
# name参数将该Container命名为demo.mysql
# p参数将Container内部的3306端口（MySQL的默认端口）映射到本地
# e参数添加环境变量，这里将MySQL的root用户密码设置为password
# d参数指定该Container在后台允许，即不会在当前命令行中显示输出
docker run --name demo.mysql -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d mysql
```

如果一切正常的话，命令行中将只输出Container的ID，同时我们也可以通过`docker ps`命令查看到。

![docker-1-0](../Images/docker-1-0.png)

如果有更进一步的需要，还可以直接进入Container内部操作（可以视作Lunix环境）。

```powershell
docker exec -it demo.mysql bash
```

![docker-1-1](../Images/docker-1-1.png)

## 关于数据卷

数据卷(volume)是一个生命周期独立于容器(container)的特殊目录，可以实现数据的持久化储存。

说到这里，想必大家应该都能想到，MySQL这样的数据库应用必然会需要建立数据卷。我们可以用`docker volume ls`来查看一下。

![docker-1-2](../Images/docker-1-2.png)

果然，MySQL镜像在创建实例容器时，会默认创建一个数据卷。

因此，这就涉及到一个坑。
在删除MySQL的实例时，我在一开始就只使用`docker rm demo.mysql`。

但是，这个操作并不会删除与容器相关的数据卷。

所以，这里需要注意，如果确定不再需要了，可以通过`docker rm -v demo.mysql`来彻底删除（包括数据卷）。

当然，对于已经存在的且不再需要的数据卷，我们也可以通过以下方式来删除。

```powershell
# 删除某个数据卷
docker volume rm bb338dbf12e4fff1deae0260ab55089a53555a6e340ae2d2f823920e7be3d725a
# 删除所有数据卷
docker volume rm $(docker volume ls -q)
```

这里稍微解释一下删除所有数据卷的命令。
它其实包括了两部分

1. docker volume ls -q
2. docker volume rm

其中的第一部分可以列出所有数据卷的Id，然后将值传给第二个命令，即起到了删除所有数据卷的作用。