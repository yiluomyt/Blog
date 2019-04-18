# ABP 框架踩坑记录

[ASP.NET Boilerplate](https://aspnetboilerplate.com/)是一个专用于现代 Web 应用程序的通用应用程序框架。

它使用了你已经熟悉的工具，并根据它们实现最佳实践。

## 开始前的准备

此处使用的为 ABP Module Zero，ASP.NET Core + Vue.js，版本为 4.4。

开发环境：

- Visual Studio 2017 Community
- .NET Core 2.2

可以在此下载初始模板: [https://aspnetboilerplate.com/Templates](https://aspnetboilerplate.com/Templates)

![下载初始模板](../Images/ABP框架入门踩坑/下载初始模板.png)

在 Choose your project's name 中填写你的项目名称，例如我这里用的是`Qincai`。

解压后打开，aspnet-core 文件夹下的 sln 文件即可看到如下目录：

![解决方案目录](../Images/ABP框架入门踩坑/解决方案目录.png)

## 文章说明

1. 根据大家键入的项目名不同，生成的模板代码也会有所区别，在文章中我就以我的`Qincai`为例，不再做具体说明，大家如果发现找不到文件或者类，脑补替换为你的项目名即可。
2. 本系列文章不作为教程，只是记录学习 ABP 框架过程中遇到的一些问题。对于每一个问题，我会先给出问题的起因和解决方案，并尽可能的还原我解决问题的步骤。

   > 作为一个在不断更新的框架，我认为单纯给出某一版本的解决方案只能解一时之急，一旦版本更新，很多问题的解决方案都有可能发生变化。所以给出解决问题的步骤，也许更能帮助大家解决问题并了解这个框架。

3. 作为初学者很多问题的解决方案也许还欠考虑，希望能与大家进一步交流。

   > 邮箱: [mytyiluo@outlook.com](mailto:mytyiluo@outlook.com)
