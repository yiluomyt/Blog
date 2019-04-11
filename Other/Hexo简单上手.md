---
tags:
  - Hexo
  - Blog
date: 2019-02-10
---

# Hexo 简单上手

说会静态博客，其实像我这样的用 Gitbook 来展示的可能还是少数，毕竟更多的情况下，还是希望自己的博客更易检索，或是更美观。

那这就不得不提到 Hexo 了，作为静态博客中的主流生成器，Hexo 无论是在生态，或是拓展性上都能让我满意。
在这就简单记录一下，我的上手体验。

## 准备阶段

首先，还是应该简单的介绍一下 Hexo。

Hexo 是一款基于 Node.js 开发的博客框架，可将 Markdown 文本渲染成静态的 html 网页。

官网有丰富的[中文文档](https://hexo.io/zh-cn/docs/index.html)，因此不再赘述安装过程，推荐在完成官网上的[建站教程](https://hexo.io/zh-cn/docs/setup)之后继续阅读。

## NexT 主题

如果在此外看过有关 Hexo 的教程的话，就会知道下一步该安装主题了。这也就是 Hexo 强大生态的体现了，一般情况下你都可以在[官方推荐列表](https://hexo.io/themes/)中找到你中意的主题。

这里我选用的[NexT 主题](https://theme-next.org/)，黑白界面非常干净，而且拓展性强。

![Hexo主题预览](https://theme-next.org/images/docs/next-schemes-3.png)

接下来，我就简单记录一些初步搭建的过程。

## 构建博客

```shell
# 初始化Hexo，demo为你的文件夹名
hexo init demo
cd demo
# 下载NexT主题
git clone https://github.com/theme-next/hexo-theme-next themes/next
# (可选)删除默认主题
rm themes/landscape -r
# 打开VS Code开始编辑
code .
```

首先，我们来修改`_config.yml`配置文件，找到`theme`字段，将其改为我们现在的主题`next`。

现在就可以通过`hexo s`来启动我们的博客了。

> 这里暂且略过一些较为简单的部分。

---

## 遇到的一些问题

### 问题一：如何置顶一篇博客

> 参考资料: [解决 Hexo 置顶问题](http://www.netcan666.com/2015/11/22/%E8%A7%A3%E5%86%B3Hexo%E7%BD%AE%E9%A1%B6%E9%97%AE%E9%A2%98/)

原本 NexT 是支持置顶功能的，但因为[一些 Bug](https://github.com/iissnan/hexo-theme-next/issues/415)后来被取消了。

那我们就从 Hexo 本身来找解决方案，文章的排序是由`hexo-generate-index`这个组件决定的，其中`lib/generator.js`中这样一行代码：

```js
var posts = locals.posts.sort(config.index_generator.order_by);
```

那我们就可以通过改写排序规则来实现置顶功能，这里我们可以在文章中定义一个`sticky`字段，其值大小决定置顶的顺序。

借助`thenby`库，我们的排序逻辑可以写为：

> 需要安装`thenby`库，npm install thenby 或者 yarn add thenby

```js
var firstBy = require("thenby");

// ...

// 配合next主题使用关键词sticky
// updated 表示更新时间，默认为创建时间
var posts = locals.posts.data.sort(firstBy("sticky", -1).thenBy("updated", -1));
```

不足：**需要手动修改源代码，对 CI/CD 不友好，有待后期改进**

## 问题二：设置分享时的摘要内容

在分享文章时，浏览器往往根据网页的元数据来提取关于网页的描述信息，例如标题、简介、图片等，用以在社交软件中形成分享卡片的形式。

通过查找资料，我们可以在`layout/_custom/head.swig`中添加如下信息来定义元数据：

{% raw %}

```html
<meta itemprop="name" content="{{ page.title | default(title) }}" />
<meta itemprop="image" content="{{ url_for('/uploads/logo-white.png') }}" />
<meta
  name="description"
  itemprop="description"
  content="{{ truncate(strip_html(page.excerpt), {length: 20}) | default(description) }}"
/>
```

{% endraw %}

其中所涉及到的变量可以在[文档](https://hexo.io/zh-cn/api/locals)中找到。

然后，我们需要转到`layout/_layout.swig`文件，修改如下内容：

{% raw %}

```html
<html class="{{ html_class | lower }}" lang="{{ config.language }}">
  <head>
    <!-- 这里将cache改为false -->
    {{ partial('_partials/head/head.swig', {}, {cache: false}) }} {% include
    '_partials/head/head-unique.swig' %}
    <title>{% block title %}{% endblock %}</title>
    {% include '_third-party/analytics/index.swig' %} {{
    partial('_scripts/noscript.swig', {}, {cache: theme.cache.enable}) }}
  </head>
</html>
```

{% endraw %}

> 在`hexo g`命令中，hexo 会对部分内容做缓存处理，而`hexo s`不会，这点坑了我不少时间。
