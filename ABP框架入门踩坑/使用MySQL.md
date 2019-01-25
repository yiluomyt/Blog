# 使用MySQL

## 起因

因为我自用的服务器只是腾讯云1核1G的学生机，不方便装SQL Server，所以转而MySQL。

> 这里使用的MySQL版本号为 8.0。

## 解决方案

1. 删除`Qincai.EntityFrameCore`项目中现有的针对SQL Server的迁移文件，即`Migrations`文件夹。
2. 为`Qincai.EntityFrameCore`项目，添加`Pomelo.EntityFrameworkCore.MySql`NuGet引用，并删除对`Microsoft.EntityFrameworkCore.SqlServer`的引用。

    > Orcale官方也有提供`MySql.Data.EntityFrameworkCore`，但在之前我使用的时候（18年10月）还存在一些Bug，不知道现在有没有修复。如果有知道的同学，可以告知我一下。
3. 在`Qincai.Web.Host`项目中的`appsettings.json`中修改连接字符串。

    ![修改连接字符串](../Images/ABP框架入门踩坑/使用MySQL/修改连接字符串.png)

    例如这里，是我在本地由Docker启动的MySQL。
4. 找到`Qincai.EntityFrameCore`项目下的`QincaiDbContextConfigurer.cs`文件，修改两处注释的地方。

    ```csharp
    using System.Data.Common;
    using Microsoft.EntityFrameworkCore;

    namespace Qincai.EntityFrameworkCore
    {
        public static class QincaiDbContextConfigurer
        {
            public static void Configure(DbContextOptionsBuilder<QincaiDbContext> builder, string connectionString)
            {
                //builder.UseSqlServer(connectionString);
                builder.UseMySql(connectionString);
            }

            public static void Configure(DbContextOptionsBuilder<QincaiDbContext> builder, DbConnection connection)
            {
                //builder.UseSqlServer(connection);
                builder.UseMySql(connection);
            }
        }
    }
    ```
5. 如下图添加Migration。
    ![添加Migration](../Images/ABP框架入门踩坑/使用MySQL/添加Migration.png)

    这里需要注意的是，默认项目必须修改为`Qincai.EntityFrameworkCore`项目，并且你解决方案的启动项目需要设置为`Qincai.Web.Host`项目。
6. 然后，就正常`Update-Database`完事了。

## 经历

最开始，要换数据库嘛，先查了下官网[这篇流程](https://aspnetboilerplate.com/Pages/Documents/EF-Core-MySql-Integration)，然后其实就差不多了，过程很简单。

而在这过程中，可能大家会看到类似这样的提示。

![错误提示](../Images/ABP框架入门踩坑/使用MySQL/错误提示.png)

就如同其字面意思，在新版的SDK中已经包含了这些工具。如果觉得看得不爽，在对应的`.csproj`文件中找到类似下方的代码，删除即可。

```xml
<ItemGroup>
<DotNetCliToolReference Include="Microsoft.EntityFrameworkCore.Tools.DotNet" Version="2.0.0" />
</ItemGroup>
```