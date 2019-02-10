# 使用Git子模块管理Hexo博客

在使用Hexo时，我们往往会clone一些主题，甚至一些主题内还包含一些子模块，例如NexT。这么做的初衷很好，但又引来了新的问题，我们该如何保存我们的博客源文件呢？

直接在`.gitignore`中屏蔽themes文件夹是一种解决方案，但在你另一个地方clone博客的时候就不得不手动加载原有的主题，更别说还要对主题进行一些自定义修改。

最终，我采用Git子模块的方式来解决多储存库关联的问题。

> 官方文档：[Git Submodules](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

就以我此前构建的博客为例，其目录结构大致如下：

```txt
Hexo
├─...
└─themes
  └─next
    ├─...
    └─source
      ├─...
      └─lib
        ├─canvas-nest
        ├─fancybox
        ├─jquery_lazyload
        ├─pace
        └─...
```

首先，我们将`themes/next`配置为`Hexo`的子模块。

为配置好后的NexT主题建立一个储存库，也可以存放在博客储存库的不同分支上。

然后，在Hexo根目录下建立`.gitmodules`文件：

```ini
[submodule "theme"]
path = themes/next
url = 你的储存库链接
branch = 默认为master
```

保存后，若你使用的是VS Code，应该就可以看到next这个文件夹被打上了`S`标记。

![next子模块](../Images/Other/使用Git子模块关联Hexo博客/next子模块.png)

同理，在next文件夹下建立`.gitmodules`文件：

```ini
[submodule "canvas-nest"]
path = source/lib/canvas-nest
url = https://github.com/theme-next/theme-next-canvas-nest.git

[submodule "fancybox"]
path = source/lib/fancybox
url = https://github.com/theme-next/theme-next-fancybox3

[submodule "jquery_lazyload"]
path = source/lib/jquery_lazyload
url = https://github.com/theme-next/theme-next-jquery-lazyload

[submodule "pace"]
path = source/lib/pace
url = https://github.com/theme-next/theme-next-pace
```

最后保存提交即可。

---

在clone该项目时，我们可以添加`--recursive`参数来为我们自动解析项目中包含的子模块。