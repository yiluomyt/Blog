# Docker部署

虽然.net core的环境配置足够简单，但我们还是希望最好能通过一键式的命令来进行发布和升级。在我已知的解决方案中，Docker是最适合这类情节的了。（若大家有其他方案，欢迎指教）

关于ASP.NET Core如何构建Docker镜像，我已经在[另一篇文章](https://gitbook.mytyiluo.cn/Docker/容器化ASPNETCore应用.html)中做了相关的介绍，这里就不再赘述了。

> 这篇我暂时没打算逐步介绍流程，还是先记录几个当前项目中遇到的坑。

## 数据库迁移(Database Migration)

对于ASP.NET Core的开发来说，使用数据库的最简单方式也就是EF Core。但EF的Code First模式，会要求启动应用前对数据库应用迁移来实现架构的统一。

目前，因为在测试阶段，我们计划是将web应用和mysql数据库同时用docker-compose来启动。所以，要确保mysql在web访问时切实可用，我们需要确保在web应用启动前已经完全迁移。
> 这里只是在测试阶段的方案，也就是说，每次启动数据库应该都只包含种子数据。

1. 导出SQL文件

    `dotnet ef migrations script`（或者在程序包管理器中使用`Script-Migration`）

    另外亦可使用，`-o`参数来指定输出文件。
2. 挂载SQL文件

    在MySQL容器启动时，在数据库为空时会自动加载`/docker-entrypoint-initdb.d/`路径下的sql文件以及包含sql文件的压缩包。

    因此，我们只需添加启动参数`-v ./init.sql:/docker-entrypoint-initdb.d/init.sql`，即可实现。

    > 若需要保留数据，可以将MySQL容器中的数据库文件映射出来，`-v ./mysql:/var/lib/mysql`。
    >
    > 在启动时数据库不为空的情况下，不会重新加载`/docker-entrypoint-initdb.d/`中的sql。

我所使用的docker-compose配置

> 当然不能用于生产环境，这边为了方便测试就直接用root用户了。
>
> 其中，配置数据库字符集的`charset.cfg`文件[见此](https://gitbook.mytyiluo.cn/Docker/使用MySQL.html#关于数据库字符集)。

```yaml
version: '3'
services:
  mysql:
    container_name: db
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=web
    volumes:
      - ./mysql:/var/lib/mysql
      - ./charset.cnf:/etc/mysql/conf.d/charset.cnf
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  web:
    depends_on:
      - mysql
    container_name: web
    image: web:1.0
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings:MySQL=server=mysql;userid=root;pwd=password;port=3306;database=web;Charset=utf-8;
```

## 镜像文件的上载

镜像文件的上载其实其实并不是一个问题，直接使用Registry就可以很好的解决该问题。在国内，像腾讯云目前就提供了免费的镜像仓库。但是，当我们服务器在海外，而不便拉取国内镜像时，我们可以在服务器端创建一个临时的私有仓库来实现镜像的上载。

1. 拉取官方的registry镜像

    `docker pull registry`
2. 启动容器

    `docker run -d -p 5000:5000 --restart always --name registry registry`
3. 在本地为镜像添加新的标签

    `docker tag {image} {domain:port}/{image}`

    > 若服务器没有配置域名的话，亦可用服务器IP替代。
4. 从本地直接向服务器推送镜像

    `docker push {domain:port}/{image}`