# 使用MySQL

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
docker run --name demo.mysql -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d mysql:5.7
```

> 注意指定版本为5.7，最新版本的lastest以更新为8.0，很多地方不兼容。💀

如果一切正常的话，命令行中将只输出Container的ID，同时我们也可以通过`docker ps`命令查看到。

![启动MySQL](../../Images/Docker/Docker入坑系列/使用MySQL/启动MySQL.png)

如果有更进一步的需要，还可以直接进入Container内部操作（可以视作Lunix环境）。

```powershell
docker exec -it demo.mysql bash
```

![MySQL命令行](../../Images/Docker/Docker入坑系列/使用MySQL/MySQL命令行.png)

## 关于数据卷

数据卷(volume)是一个生命周期独立于容器(container)的特殊目录，可以实现数据的持久化储存。

说到这里，想必大家应该都能想到，MySQL这样的数据库应用必然会需要建立数据卷。我们可以用`docker volume ls`来查看一下。

![数据卷](../../Images/Docker/Docker入坑系列/使用MySQL/数据卷.png)

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

## 创建MySQL集群

如果觉得只是启动一个MySQL体现不出Docker的优势，那么这边我们来试着用Docker创建一个MySQL集群。

### 使用命令行创建

首先，自然是[官方文档](https://hub.docker.com/r/mysql/mysql-cluster/)。

因为官方文档已经讲的挺详细的了，所以这边只是单纯的题一下。

```PowerShell
# 创建一个内部网络,命名为cluster
docker network create cluster --subnet=192.168.0.0/16
# 创建MySQL管理节点
docker run -d --net=cluster --name=management1 --ip=192.168.0.2 mysql/mysql-cluster ndb_mgmd
# 创建两个MySQL数据节点
docker run -d --net=cluster --name=ndb1 --ip=192.168.0.3 mysql/mysql-cluster ndbd
docker run -d --net=cluster --name=ndb2 --ip=192.168.0.4 mysql/mysql-cluster ndbd
# 创建MySQL查询节点
# 这里和官方文档不同，为了方便直接设置了root密码
docker run -d --net=cluster --name=mysql1 --ip=192.168.0.10 -e MYSQL_ROOT_PASSWORD=password mysql/mysql-cluster mysqld
```

![MySQL集群](../../Images/Docker/Docker入坑系列/使用MySQL/MySQL集群.png)

此时，MySQL集群就已经启动了，我们可以利用`ndb_mgm`来管理集群。

```PowerShell
# 进入交互式的管理命令行
docker run -it --net=cluster mysql/mysql-cluster ndb_mgm
```

在显示`ndb_mgm>`后键入`show`就可以看到该集群信息。

![集群信息](../../Images/Docker/Docker入坑系列/使用MySQL/集群信息.png)

## 使用Docker Compose创建

和网上那些一大堆的配置教程对比是不是简单很多，但也还是有些不爽，特别是在你要清理资源的时候，你需要手动的删除容器、网络、数据集资源。

显然，我们有更简洁的方式来启动MySQL集群——利用Docker Compose。（懒得换顺序了，本来docker-compose是在下一篇中介绍的）

那首先，我们需要新建一个目录，比如**mysql_cluster**。（目录名称和默认的容器名称有关）

然后，在这个目录中创建一个配置文件`docker-compose.yml`，键入以下内容。

```yml
version: '3'

services:
  # 管理节点
  management:
    image: "mysql/mysql-cluster"
    command: ndb_mgmd
    networks:
      cluster:
        ipv4_address: 192.168.0.2

  # 数据节点1
  ndb1:
    image: "mysql/mysql-cluster"
    command: ndbd
    networks:
      cluster:
        ipv4_address: 192.168.0.3

  # 数据节点2
  ndb2:
    image: "mysql/mysql-cluster"
    command: ndbd
    networks:
      cluster:
        ipv4_address: 192.168.0.4

  # 查询节点
  mysql:
    image: "mysql/mysql-cluster"
    command: mysqld
    # 设置root密码
    environment:
      - MYSQL_ROOT_PASSWORD=password
    # 将3306端口暴露到本地
    ports:
      - "3306:3306"
    networks:
      cluster:
        ipv4_address: 192.168.0.10

# 创建一个B类子网
# 其实改成A类网络应该也可以，这里就按官方文档来配置了。
networks:
  cluster:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.0.0/16
```

最后，在该目录下启动PowerShell，键入`docker-compose up -d`，即可启动MySQL集群。

### MySQL集群在测试时的一些小问题

在创建MySQL集群后，相信大家肯定会用各种工具去连接测试一下，但往往并不会顺利的连接上。

比如，极有可能碰到的权限问题。
> host ‘192.168.0.1’ is not allowed to connect to this MYSQL server.

这是因为我们的MySQL集群运行在子网中，我们在本地环境中去访问的话，就相当于远程访问。但是MySQL的默认策略是不允许远程登录的，所以我们需要进入MySQL容器，添加远程访问的权限。

首先，利用`docker exec -it mysqlcluster_mysql_1 mysql -uroot -p`进入查询节点。

> **注意**:这里的容器名是根据我创建的目录`mysql-cluster`生成的默认命名。若你创建的目录名和我不同，请使用`docker ps`来查看你自己查询节点的容器名。

然后，在验证密码之后，键入以下内容。

```sql
/*
* 这条命令是赋予任何IP的root用户全部权限
* 其中，root用户的密码为password
*/
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
```

然后，重新访问就OK了。