---
tags:
  - ASP.NET Core
  - ABP
date: 2019-01-25
---

# 配置 User Secrets

## 起因

因为以往习惯在 User Secrets 中保存连接字符串之类信息，但当我把连接字符串移到 secrets.json 中后，却发现在迁移过程中会报如下的错误：

![迁移报错](../Images/ABP框架入门踩坑/配置UserSecrets/迁移报错.png)

简单说，也就是迁移时无法获取到连接字符串信息。

## 解决方案

1. 在 Qincai.EntityFrameworkCore 项目中，找到 QincaiDbContextFactory.cs 文件，修改如下注释处代码。

   ```csharp
   public class QincaiDbContextFactory : IDesignTimeDbContextFactory<QincaiDbContext>
   {
       public QincaiDbContext CreateDbContext(string[] args)
       {
           var builder = new DbContextOptionsBuilder<QincaiDbContext>();
           //var configuration = AppConfigurations.Get(WebContentDirectoryFinder.CalculateContentRootFolder());
           var configuration = AppConfigurations.Get(WebContentDirectoryFinder.CalculateContentRootFolder(), addUserSecrets: true);

           QincaiDbContextConfigurer.Configure(builder, configuration.GetConnectionString(QincaiConsts.ConnectionStringName));

           return new QincaiDbContext(builder.Options);
       }
   }
   ```

## 经历

这个问题看似解决容易，但还是费了我不少时间才找到原因。

首先，我怀疑是不是 secrets.json 文件有问题，就把 Qincai.Web.Host.csproj 中的 UserSecretsId 删掉，重新生成了一个。但难受的是，这下不单是迁移有问题了，连应用都无法启动了，当然报错信息也是无法找到连接字符串。

所以，我开始在 StartUp 中找注入配置的代码，然后发现不同于微软模板代码中直接注入 IConfiguration 对象的做法，Module Zero 自己实现了一个拓展方法 IHostingEnvironment.GetAppConfiguration。其源码如下：

```csharp
public static IConfigurationRoot GetAppConfiguration(this IHostingEnvironment env)
{
    // 这里第三个参数代表是否添加User Secrets
    // 可以看到Module Zero默认只在开发环境中添加
    return AppConfigurations.Get(env.ContentRootPath, env.EnvironmentName, env.IsDevelopment());
}
```

这里调用了 AppConfigurations 类，**注意这个类是定义在 Qincai.Core 项目中**，这是导致问题的关键。

然后，我们来研究一下 AppConfigurations 类中的这段代码：

```csharp
private static IConfigurationRoot BuildConfiguration(string path, string environmentName = null, bool addUserSecrets = false)
{
    var builder = new ConfigurationBuilder()
        .SetBasePath(path)
        .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);

    if (!environmentName.IsNullOrWhiteSpace())
    {
        builder = builder.AddJsonFile($"appsettings.{environmentName}.json", optional: true);
    }

    builder = builder.AddEnvironmentVariables();

    if (addUserSecrets)
    {
        // 在这里添加了User Secrets
        builder.AddUserSecrets(typeof(AppConfigurations).GetAssembly());
    }

    return builder.Build();
}
```

乍一看好像没问题，但是注意，我刚才说了 AppConfigurations 是属于 Qincai.Core 项目，即根据默认的配置，typeof(AppConfigurations).GetAssembly()获取到的是 Qincai.Core.dll 下的程序集。

也就是说，这里添加的 User Secrets 是根据 Qincai.Core.csproj 中定义的 UserSecretsId，而不是 Qincai.Web.Host.csproj 中定义的。

打开 Qincai.Core.csproj，我们确实注意到有一个 UserSecretsId 字段，且和 Qincai.Web.Host.csproj 中未修改器前的一致。

这里就是 Module Zero 取巧的地方了，因为在 VS 中，只有 Web 项目才能在右键中找到**管理用户机密**，所以通过两个配置两个相同的 UserSecretsId，使其能通过 VS 的快捷方式修改 Qincai.Core 项目的 secrets.json 文件。

既然找到原因了，那么只需将 UserSecretsId 恢复成一致即可，痛快地敲下 F5，启动成功。

因此**注意**，一旦在你修改了 Qincai.Web.Host.csproj 中的 UserSecretsId 后，千万不要忘了修改 Qincai.Core.csproj，务必确保两个 UserSecretsId 一致，不然你再怎么改，程序也是届不到的。

---

> 不是完了吗？才怪嘞！！我们的问题是迁移时无法读取 User Secrets，以上经历只能说又回到了起点。

到这里，我们已经可以在程序运行时成功读取到 User Secrets，但是在数据库迁移过程中，还是会报错:

```shell
System.ArgumentNullException: Value cannot be null.
Parameter name: connectionString
```

让我们打开 ef 工具的[详细输出`-v`](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dotnet#common-options)来看一看，其中有这么一段输出：

```shell
Finding DbContext classes...
Finding IDesignTimeDbContextFactory implementations...
Finding application service provider...
Finding IWebHost accessor...
Using environment 'Development'.
Using application service provider from IWebHost accessor on 'Program'.
Finding DbContext classes in the project...
Found DbContext 'QincaiDbContext'.
Using DbContext factory 'QincaiDbContextFactory'.
```

它提到了 QincaiDbContextFactory 这个类，源码如下：

```csharp
/* This class is needed to run "dotnet ef ..." commands from command line on development. Not used anywhere else */
public class QincaiDbContextFactory : IDesignTimeDbContextFactory<QincaiDbContext>
{
    public QincaiDbContext CreateDbContext(string[] args)
    {
        var builder = new DbContextOptionsBuilder<QincaiDbContext>();
        // 注意这里
        var configuration = AppConfigurations.Get(WebContentDirectoryFinder.CalculateContentRootFolder();

        QincaiDbContextConfigurer.Configure(builder, configuration.GetConnectionString(QincaiConsts.ConnectionStringName));

        return new QincaiDbContext(builder.Options);
    }
}
```

看到注释，我们应该就是找对地方了，注意我在代码中标出的位置。它也通过 AppConfigurations.Get 来获取配置，但是没有给出 AddUserSecrets 参数（默认为 false），而根据此前的代码可知，它没有添加 User Secrets。

那么解决方案就很简单了，显式给出 AddUserSecrets 参数即可。

```csharp
var configuration = AppConfigurations.Get(WebContentDirectoryFinder.CalculateContentRootFolder(), addUserSecrets: true);
```
