# seafile服务器迁移

## 起因

最初的seafile是搭建在1核1G1Mbps的服务器上，普通使用虽然还是没问题的，但在一些需要分享的时候就会显得配置不足，尤其是带宽，在有多个用户同时下载的时候，网速一度只有70+kb/s。

基于此问题，再考虑到实际的应用场景（每个月用不了几次），最后，还是考虑将服务器升级一下（主要是内存），随便把网络改为按流量计费。

## 数据备份

首先第一步，还是要先把数据备份一下以防万一。seafile的数据主要有两部分，资源文件和数据库。

其中，因为我使用的是Docker版本进行部署，所以只需将挂载点下的文件都复制出来即可。

对于数据库，在[官方文档](https://manual-cn.seafile.com/maintain/backup_recovery.html#%E5%A4%87%E4%BB%BD%E6%95%B0%E6%8D%AE%E5%BA%93)中找到了相应的操作，只需备份以下三张表即可。

```bash
mysqldump -uroot --opt ccnet_db > ccnet_db.sql
mysqldump -uroot --opt seafile_db > seafile_db.sql
mysqldump -uroot --opt seahub_db > seahub_db.sql
```

## 新服务器的准备工作

在正式开始迁移之前，还是需要将新购服务器上的环境配置一下。

> 新服务器的配置如下，1核4G10Mbps，系统为centos 7.5，50G系统盘再加50G数据盘以便拓展。

由于计划将seafile的数据存放到独立的云硬盘上，所以，我们需要先将云硬盘格式化并挂载到服务器上。

### 挂载云硬盘

> 此处步骤可参考腾讯云教程 - [Linux 系统分区、格式化、挂载及创建文件系统](https://cloud.tencent.com/document/product/362/6735)。

首先，让我们查看一下当前系统已挂载的硬盘：`fdisk -l`。

![查看已挂载硬盘](../Images/Other/seafile服务器迁移/查看已挂载硬盘.png)

根据上图，我们可以看到当前主机上挂载了两块硬盘：

- `/dev/vda`为系统盘。
- `/dev/vdb`为额外的数据盘。

下一步，我们需要确认云硬盘的文件系统：`file -s /dev/vdb`。
> `/dev/vdb` 来自上一命令中显示的数据盘。

若得到以下的输出，说明该硬盘还未格式化。（若是新购的云硬盘应该都是未格式化的）

```bash
# file -s /dev/vdb
/dev/vbd: data
```

对于这样未格式化的硬盘，我们首先需要为其创建文件系统：`mkfs.ext4 /dev/vdb`。
> 这里我创建的是ext4文件系统，若想创建其他文件系统可以参考腾讯云所给教程。

创建完成后，再次确认文件系统，若得到以下输出说明创建成功。

![确认文件系统](../Images/Other/seafile服务器迁移/确认文件系统.png)

然后，我们需要创建一个文件目录，用于挂载硬盘：`mkdir /data -p`。

并将该硬盘挂载到刚创建的目录下：`mount /dev/vdb /data`。

### （可选）设置自动挂载

1. 备份默认文件
    `cp /etc/fstab /etc/fstab.backup`
2. 添加以下内容

    `device_name  mount_point  file_system_type  fs_mntops  fs_freq  fs_passno`

    其中`fs_mntops` `fs_freq` `fs_passno`分别取`defaults,nofail` `0` `1`。

    此外，
    - `device_name` 通过 `ls -l /dev/disk/by-id/` 确定；
    - `mount_point` 表示挂载点；
    - `file_system_type` 通过 `file -s device_name` 确定。
3. 测试是否成功
    `mount -a` 若无错误即可。

![编辑配置文件](../Images/Other/seafile服务器迁移/编辑配置文件.png)

### 安装Docker环境

首先，保持一个好习惯，上手先更新一波yum：`yum upadte -y`。

随后安装Docker的依赖包：

```bash
yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2
```

考虑到国内环境对官方源不怎么友好，这里我们使用阿里云的源。

```bash
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

在添加新的软件源之后，再更新下yum缓存，确保已经更新引用：`yum makecache fast`。

最后，安装Docker-ce反而显得简单：`yum install -y docker-ce`。

### 配置Docker

和上面一样的原因，我们也需要将Docker镜像源替换为国内的镜像站。

修改/etc/docker/daemon.json文件，添加以下内容：

```json
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

然后，启动Docker服务：`systemctl start docker`。
> 若在配置镜像站之前已经启动了Docker服务，则需重新启动：`systemctl restart docker`。

测试Docker是否可以正常使用：`docker run hello-world`，若看到熟悉的输出即成功了。

## 安装与迁移

### 安装seafile

下载seafile镜像：`docker pull seafileltd/seafile`。

启动seafile

```bash
docker run -d --name seafile \
  -e SEAFILE_SERVER_HOSTNAME=yourdaemon \
  -e SEAFILE_ADMIN_EMAIL=admin@email \
  -e SEAFILE_ADMIN_PASSWORD=password \
  -v /data/seafile-data:/shared \
  -p 8080:80 \
  seafileltd/seafile:latest
```

这里值得注意的有两个参数：

1. `-v /data/seafile-data:/shared`
    将数据盘下的目录挂载到容器中。
2. `-p 8080:80`
    将内部的80端口暴露到本地的8080端口。
    > 在我们使用的镜像版本中，官方在内部配置了Nginx反向代理，通过80端口访问。

### 迁移数据

首先，将之前备份的数据文件和sql文件复制到宿主机的对应目录下。

然后，进入容器：`docker exec -it seafile bash`。

还原数据库

```bash
mysql -uroot ccnet_db < ccnet_db.sql
mysql -uroot seafile_db < seafile_db.sql
mysql -uroot seahub_db < seahub_db.sql
```

cd到seafile-server-latest下，运行垃圾清理

```bash
bash seafile.sh stop
bash seaf-gc.sh
bash seafile.sh start
```

## 遗留问题

1. 迁移后，头像和自定义的图标会发生错误。
    > 目前只想到重新上传来解决。