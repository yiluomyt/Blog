---
tags:
  - ASP.NET Core
date: 2018-05-04
---

# 最简单的 CURD

对于任何一个 WEB 应用来说，CRUD 都是基础中的基础，虽说每本书上都会说，但是对于 Asp.Net Core 来说，还是有一些值得注意的点。

> CRUD: create, read, update, delete 的首字母缩写

本文对应[Demo](https://github.com/panfengstudio/workshop/tree/2018/05/05)。

## EntityFramework Core

Doc: [EF Core](https://docs.microsoft.com/zh-cn/ef/core/)

EF Core 随着.net core 的版本更新，现在也是到了 2.0 版本，坑也少了不少。根据张队最近的[调查报告](https://mp.weixin.qq.com/s?__biz=MzAwNTMxMzg1MA==&mid=2654070376&idx=1&sn=cf613167592f9955dd5b76424a430dc6)来看,EF Core 的使用率也不低。

当然，作为一个没写过大型项目的新人，没资格去讨论什么性能、优化之类，只能说 EF Core 对小型的 WEB 应用，以及新人上手还是非常友好的，仅需通过简单的配置，就可以像使用内存中对象一样操纵数据库。

查询

```csharp
// 使用基于Method的LINQ
var records = _context.Records
    .Where(record   => record.SignInTime >= begin && record.SignInTime <= end)
    .Select(record  => new
    {
        StudentId   = record.StudentId,
        Name        = record.Name,
        SignInTime  = record.SignInTime,
        SignOutTime = record.SignOutTime
    }).ToList();

// 使用基于Query的LINQ
Record record = (from r in _context.Records
                where r.StudentId == model.StudentId
                && r.SignOutTime == null
                select r).FirstOrDefault();
```

写入

```csharp
// 根据模型数据创建记录
Record record = new Record
{
    StudentId  = model.StudentId,
    Name       = model.Name,
    SignInTime = DateTime.Now
};
// 向上下文中添加记录，并保存
_context.Add(record);
_context.SaveChanges();
```

修改

```csharp
// 记录签到时间并保存
record.SignOutTime = DateTime.Now;
_context.Update(record);
_context.SaveChanges();
```

可以看到在这些简单的例子中，每次修改上下文后，都需要`SaveChanges`才能将修改保存到数据库，对上下文的修改仅是保存一个标记。

## 依赖注入

Doc: [依赖注入](https://docs.microsoft.com/zh-cn/aspnet/core/mvc/controllers/dependency-injection)(DI)

这应该是 Asp.Net Core 中最显著的特征了，所有的依赖都是通过 IOC 框架来提供，同时微软也在 Asp.Net Core 中提供了一个够用的官方 IoC 框架。

微软的推荐实践是在 StartUp.cs 的 ConfigureServices 中添加相关依赖。

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // 添加内存数据库
    services.AddDbContext<RecordDbContext>(options =>
        options.UseInMemoryDatabase("WorkShopDev"));
    // 添加MVC服务
    services.AddMvc();
}
```

然后，通过构造函数注入。(当然，在有需要时，也可以在对应函数中注入)

```csharp
public RecordController(RecordDbContext context)
{
    _context = context;
}
```

## 模型绑定

Doc: [模型绑定](https://docs.microsoft.com/zh-cn/aspnet/core/mvc/models/model-binding)(Model Binding)

Asp.Net Core 会用一种 Magic 的方法去匹配你所需要的对应参数，分别从以下几个数据源（优先级从上到下）：

1. Form values：这些是使用 POST 方法进入 HTTP 请求的窗体值。 （包括 jQuery POST 请求）。
2. Route values：路由提供的路由值集。
3. Query strings：URI 的查询字符串部分。

如有需要，你还可以用类似`[FromForm]`这样的特性去指定某一参数的来源。

值得注意的是`[FromBody]`，由于 Body 在 Request 中是作为 Stream 存在的，所以只能读取一次，也就是说每个 Action 中至多只能出现一个`[FromBody]`。

> 当你的 Header 为`application/json`时，就需要用`[FromBody]`来绑定参数，例如微信小程序。
