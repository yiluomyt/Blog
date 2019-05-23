---
date: 2019-04-17
tags:
  - Blog
  - React
  - GatsbyJS
---

# 博客 TOC 目录

根据 Markdown 标题，我们可以生成目录来方便跳转文章中的各个章节。

这里我参考 antd 的 Anchor 和 Drawer 实现了一个简陋版本的目录，先上一下效果图：

![PC端效果图](../Images/Other/博客TOC目录/PC端效果图.png)

![移动端效果图](../Images/Other/博客TOC目录/移动端效果图.png)

## 为标题元素添加 id

首先，我们需要为标题元素加上 id，这里为了方便就使用了 gatsby-remark-autolink-headers 插件。\
若不需要其标题前的 icon，可以在 gatsby-config.js 中进行配置。

```js
{
  resolve: `gatsby-transformer-remark`,
  options: {
    plugins: [
      {
        resolve: `gatsby-remark-autolink-headers`,
        options: {
          icon: false,
        },
      },
    ]
  }
}
```

## 实现目录组件

### 目录解析

从 Gatsby 的 graphql 中获取的目录结构应为：

```
markdownRemark {
  headings {
    value
    depth
  }
}
```

一般地，博客的一级标题只有一个，且应为博客标题。所以，我们首先需要将一级标题筛选出来，为剩下的标题添加链接。

```typescript
// 一级目录
const mainHeading = headings.filter(head => head.depth === 1)[0];
// 其余目录
const links = headings
  .filter(h => h.depth > 1)
  .map(h => ({
    // 在 gatsby-remark-autolink-headers 插件中，
    // 对标题的空格用-替换，并转为小写，
    // 这里与其保持一致。
    id: h.value.replace(/ /g, "-").toLowerCase(),
    ...h
  }));
```

### 高亮当前标题

对于目录组件，我们希望其正确显示当前阅读位置，对此可以通过监听 window 的 scroll 事件实现。

```typescript
// 添加监听
componentDidMount() {
  this.scrollEvent = window.addEventListener("scroll", this.handleScroll);
  this.handleScroll();
}

// 注销监听
componentWillUnmount() {
  if (this.scrollEvent) {
    this.scrollEvent.remove();
  }
}

// 高亮当前目录
handleScroll = () => {
  this.setState({
    active: this.getCurrentAnchor(),
  });
};

// 当前的滚动距离
export function getCurrentScrollTop() {
  return (
    window.pageYOffset ||
    document.body.scrollTop ||
    document.documentElement.scrollTop
  );
}

// 计算获取当前标题
getCurrentAnchor() {
  // 添加偏移以便标题能够完整的显示
  const scrollTop = getCurrentScrollTop() + 5;
  let active: HTMLElement | null = null;
  this.links.forEach(link => {
    // 获取标题所对应的 HTML 元素
    const target = document.getElementById(link.id);
    if (target) {
      // 比较对应元素位置与当前位置
      if (target.offsetTop > scrollTop) return;
      if (!active) {
        active = target;
      }
      // 判定当前章节与当前位置
      else if (target.offsetTop >= active!.offsetTop) {
        active = target;
      }
    }
  });

  return active ? active!.id : null;
}
```

### 平滑滚动

点击标题后，我们希望页面等跳转到对应章节位置，但若直接设置滚动距离会显得较为突兀。

为了更好的体验，我们可以通过添加缓动来实现平滑滚动。

以下是借鉴自 Ant Design 的缓动函数：

```typescript
// 缓动函数
// t 当前时间
// b 当前位置
// c 目标位置
// d 总时间
function easeInOutCubic(t: number, b: number, c: number, d: number) {
  // 剩余距离
  const cc = c - b;
  // 将时间范围映射到[0, 2]
  t /= d / 2;
  // 前半段
  if (t < 1) {
    return (cc / 2) * t * t * t + b;
  }
  // 后半段
  // t -= 2 此时 t 范围为[-1, 0]
  // 故 (t -= 2) * t * t 为 y = x^3 曲线的[-1, 0]部分
  return (cc / 2) * ((t -= 2) * t * t + 2) + b;
}
```

实现的效果是，速度从慢到快再到慢，两端变化较为平缓。

借此，我们实现如下的滚动方法：

```typescript
// 平滑滚动到元素 id 所在位置
function scrollTo(id: string) {
  // 当前位置
  const scrollTop = getCurrentScrollTop();
  const target = document.getElementById(id);
  if (!target) {
    return;
  }
  // 目标位置
  const targetScrollTop = target.offsetTop;
  // 起始时间
  const startTime = Date.now();
  const frameFunc = () => {
    const timestamp = Date.now();
    // 这一帧经过时间
    const time = timestamp - startTime;
    // 根据缓动函数计算，本帧应该移动的距离
    const nextScrollTop = easeInOutCubic(time, scrollTop, targetScrollTop, 450);
    // 滚动 Y 轴
    window.scrollTo(window.pageXOffset, nextScrollTop);
    if (time < 450) {
      // 下一帧
      requestAnimationFrame(frameFunc);
    }
  };

  // 这里没有做降级处理（开发人员平时不会用IE吧！）
  requestAnimationFrame(frameFunc);
}
```

### 目录组件样式

这里我就直接贴代码了，详情请见注释。

```less
// 主题常量
@import "~@/styles/theme.less";

.catalog {
  // 适应目录高度
  height: fit-content;

  margin: 8px;

  // 一级标题
  h3 {
    padding-bottom: 4px;
    margin-bottom: 4px;
    border-bottom: 1px @color-bg solid;
  }

  // 目录标题列表
  ul {
    margin-left: 0.5em;
    list-style: none;

    li a {
      display: block;

      text-overflow: ellipsis;
      overflow: hidden;
      white-space: nowrap;
    }

    // 标题前的实心方块
    li::before {
      content: "";

      position: absolute;
      margin-top: 0.5em;
      margin-left: -1.2em;
      width: 4px;
      height: 4px;
      // 随标题元素颜色变换
      background: currentColor;
    }

    // 包含data-depth属性的li元素为目录标题
    // 默认为四级以上的标题，对这些标题不再做区分
    li[data-depth] {
      padding: 4px;
      margin-left: 3em;
    }

    // 二级标题
    li[data-depth="2"] {
      font-weight: @font-weight-semibold;
      margin-left: 0.5em;

      &::before {
        width: 6px;
        height: 6px;
      }
    }

    // 三级标题
    li[data-depth="3"] {
      margin-left: 2em;
    }

    // 标题高亮
    li:hover,
    .active {
      a {
        color: @color-primary;
      }
      background-color: @color-bg;
    }
  }
}
```

### 完整组件（见 Github）

自此就是我们目录组件的实现过程，剩余细节可以直接参考我的[源码](https://github.com/yiluomyt/MokeBlog/tree/master/src/components/catalog)。