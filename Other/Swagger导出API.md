# Swagger 导出API

## Swagger简介

Swagger是一个开源软件框架，可帮助开发人员设计，构建，记录和使用RESTful Web服务。

其中，Swagger可以生成一个交互式的API控制台，以便于快速测试API。

从个人角度讲，Swagger对于前后端分离的小团队来说是非常有帮助的。尤其是像我们这种平时没写文档习惯的人来说，Swagger能根据代码自动生成文档可谓是一大福音，再也不用被人追着问这API到底是怎么用的。

这里，我将会具体说下最近我使用Swagger的一些心得和体会，团队的开发环境如下：

- ASP.NET Core 2.1 (Visual Studio 2017 Community)
- 微信小程序 (官方工具+Visual Studio Code)

## 生成Swagger API 文档

对于ASP.NET Core来说，生成文档这一步还是相对容易的，且对代码基本没有侵入性。只需在Startup中配置一下即可，大致步骤基本如下：

### 添加NuGet包

这里，我所使用的是Swashbuckle.AspNetCore，直接用VS的NuGet包管理器下载即可。

![NuGet-Swashbuckle.AspNetCore](../Images/Other/Swagger导出API/NuGet-Swashbuckle.AspNetCore.jpg)

然后在`ConfigureServices`中配置如下:

```csharp
services.AddSwaggerGen(c =>
{
    // 定义文档
    c.SwaggerDoc("v1", new Info { Title = "Qincai API", Version = "v1" });
});
```

并在`Configure`中启用该Services：

```csharp
// 使用Swagger
app.UseSwagger();
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "Qincai API v1");
});
```

随后运行，打开浏览器`host/swagger`即可看到所生成的API文档。

![Swagger UI](../Images/Other/Swagger导出API/Swagger%20UI.jpg)

若还有不清楚的，可以参考微软官方的[文档](https://docs.microsoft.com/zh-cn/aspnet/core/tutorials/getting-started-with-swashbuckle?view=aspnetcore-2.1&tabs=visual-studio)。

完成到这里，是不是发现你和图中生成的样子有点不太一样，比如说，为什么没有注释，以及认证用的小锁，那这里我们就需要进一步配置。

### 添加XML注释

如果大家有在VS中写C#经历的话，肯定会对XML注释影响深刻，通过简单的`///`就可以自动生成规范的注释格式。那这里，我们的Swagger也正是利用了这些XML注释来标记对应的API。

首先，我们需要启用VS中导出XML文档的功能，在项目属性中，生成 > 输出，勾选XML文档文件，并填写对应的路径，我所写的是项目根目录。

![输出XML文档](../Images/Other/Swagger导出API/输出XML文档.jpg)

然后再到之前的`ConfigureServices`中添加如下：

```csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new Info { Title = "Qincai API", Version = "v1" });

    // file 是你在项目属性中配置的相对路径
    var filePath = System.IO.Path.Combine(AppContext.BaseDirectory, file);
    c.IncludeXmlComments(filePath);
});
```

重新生成，再运行，你可以看到Swagger中就对API以及参数添加上了注释。

### 添加认证功能

在实际开发中，我们常常是需要给我们的API添加上认证功能以避免非法的访问，因此我们也就需要给Swagger中的API标识上是否需要认证，并且添加提供Token的功能，以方便在Swagger的控制台中调试。

简单来说，还是在`ConfigureServies`中，配置如下：

```csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new Info { Title = "Qincai API", Version = "v1" });

    var filePath = System.IO.Path.Combine(AppContext.BaseDirectory, file);
    c.IncludeXmlComments(filePath);

    // 定义认证方式
    c.AddSecurityDefinition("Bearer", new ApiKeyScheme
    {
        In = "header",
        Description = "请键入JWT Token，格式为'Bearer '+你的Token。",
        Name = "Authorization",
        Type = "apiKey"
    });

    // 网上为全局API添加认证参数的方法
    // c.AddSecurityRequirement(new Dictionary<string, IEnumerable<string>> {
    //     { "Bearer", Enumerable.Empty<string>() },
    // });

    // 在过滤器中为需要认证的API添加对应参数
    // 过滤器的定义见下文
    c.OperationFilter<AuthorizationHeaderOperationFilter>();
});
```

这里值得说一下的是，网上普遍都是全局添加认证参数，导致一些不需要认证的API也被打上了标识，这在Swagger 控制台中影响倒不大，但在后续的导出API时就麻烦了，所有在这里，我使用自定义过滤器的方式来只为需要认证的API添加认证参数。

根据上文，我们先定义一个`AuthorizationHeaderOperationFilter`类，它需要实现`IOperationFilter`接口，类定义如下：

```csharp
/// <summary>
/// 判断是否需要添加Authorize Header
/// </summary>
public class AuthorizationHeaderOperationFilter : IOperationFilter
{
    /// <summary>
    /// 为需要认证的Operation添加认证参数
    /// </summary>
    /// <param name="operation">The Swashbuckle operation.</param>
    /// <param name="context">The Swashbuckle operation filter context.</param>
    public void Apply(Operation operation, OperationFilterContext context)
    {
        // 获取对应方法的过滤器描述
        // 应该也就是所添加的Attribute
        var filterPipeline = context.ApiDescription.ActionDescriptor.FilterDescriptors;
        // 判断是否添加了AuthorizeFilter
        // 也就是[Authorize]
        var isAuthorized = filterPipeline.Select(filterInfo => filterInfo.Filter).Any(filter => filter is AuthorizeFilter);
        // 判断是否添加了IAllowAnonymousFilter
        // 也就是[AllowAnonymous]
        var allowAnonymous = filterPipeline.Select(filterInfo => filterInfo.Filter).Any(filter => filter is IAllowAnonymousFilter);

        // 仅当需要认证且不是AllowAnonymous的情况下，添加认证参数
        if (isAuthorized && !allowAnonymous)
        {
            // 若该Operation不存在认证参数的话，
            // 这个Security将是null，而不是空的List
            if (operation.Security == null)
                operation.Security = new List<IDictionary<string, IEnumerable<string>>>();

            // 添加认证参数
            operation.Security.Add(new Dictionary<string, IEnumerable<string>>
            {
                { "Bearer", new string[] { } }
            });
        }
    }
}
```

重新生成后运行，应该就可以看到需要认证的API都带上了一把小锁。

### 注意事项

虽说配置并不算难，但还是需要注意一些地方。

1. 注意写好XML注释

    之前也说了，Swagger的注释是根据XML文档生成的，反过来说，如果你没写XML注释，Swagger上也就是不会有注释的。

    另外，在你启用XML文档输出之后，VS也会很贴心的为你把没有写XML注释的地方都标为Warning。╮(╯▽╰)╭，所以安心的把注释都补一遍吧。

2. 为参数添加数据注解

    Swagger是支持部分数据注解的，比如`[Required]`之类的。

    结合`[ApiController]`自带的模型验证功能，岂不美哉。

3. 将XML文档复制到输出目录

    若你在发布应用后发现XML不见了，那可能就是你没有为XML文档文件配置复制到输出目录的属性。

    打开资源管理器，右键对应的文件，我们在属性中可以看到有**复制到输出目录**的属性，将其设置为始终复制就Ok。

    ![XML文档属性](../Images/Other/Swagger导出API/XML文档属性.jpg)

以上就是我最近所用到的Swagger的一些功能。

## 导出微信小程序可用的API

在一开始也说了，使用Swagger的主要目的就是方便小团队的沟通，但事实上，因为我们前端的人少（大家都是CSS鬼才），导致我们开发新API的速度往往比前端进度快，没几天前端那边就需要更新一下API的库（将小程序的CallBack封装成Promise）。

因此，就有了根据Swagger自动生成Js可用的API文件的想法，其实想法的本身是来自于Abp项目的设计（真的很优秀），但出于一些方面的考虑，我们姑且还没采用Abp。

随后，就是查找资料了，确实Swagger有这方面的[支持](https://github.com/swagger-api/swagger-codegen)，其中官方[关于Js的库](https://github.com/swagger-api/swagger-js)是完全动态的，但可惜不适用于小程序。然后，就发现了第三方的[swagger-js-codegen](https://github.com/wcandillon/swagger-js-codegen)，可用于生成静态的Js代码，且提供了自定义模板的功能。网上也不少基于这个库的其他模板，比如说`axios`之类的，但没有适用于微信小程序的，不过问题不大，模板是基于`mustache`的，动手撸就是了。

模板代码有点长就不在这里放出了，大家还请移步[GitHub](https://github.com/yiluomyt/swagger-wxopen-codegen-template)。

这里我就简单说下大致思路，

模板本身是基于原来的nodejs模板改的，我们所需做的就是将http请求部分的代码改为使用`wx.request`，如下简单的封装即可:

```js
/**
* HTTP Request
* @method
* @name {{&className}}#request
* @param {string} method - HTTP 请求方法
* @param {string} url - 开发者服务器接口地址
* @param {object} data - 请求的参数
* @param {object} headers - 设置请求的 header ,默认为 application/json
*/
request(method, url, parameters, data, headers){
    return new Promise((resolve, reject) => {
        wx.request({
            url: url,
            data: data,
            header: headers,
            method: method,
            success: res => {
                if(res.statusCode >= 200 && res.statusCode <= 299) {
                    resolve(res.data)
                } else {
                    reject(res)
                }
            },
            fail: e => reject(e)
        })
    })
}
```

对应method的话，基本没怎么改，只根据微信小程序所有参数都是传递给data，做了点简化。

对于认证部分，根据我们自己的需求，换成了这样的实现：

```js
new Promise((resolve, reject) => {
    this.authenticate()
    .then(token => {
        headers['Authorization'] = 'Bearer ' + token;
        resolve(this.request('{{method}}', domain + path, parameters, data, headers))
    })
})
```

其中`this.authenticate`是由外部传入的`function`，返回一个包含`Token`的`Promise`。

导出后，在小程序中的使用就类似于:

```js
import API from './api.js'

api = new API('http://localhost:5000')
api.setAuthenticate(function () {
    return new Promise((resolve, reject) => {
        // 你的认证逻辑
        resolve(token)
    })
})
```

然后，就可以开心地调用各种方法了。

最后，再放一遍Demo的链接：[https://github.com/yiluomyt/swagger-wxopen-codegen-template](https://github.com/yiluomyt/swagger-wxopen-codegen-template)，发现有问题欢迎提Issue。