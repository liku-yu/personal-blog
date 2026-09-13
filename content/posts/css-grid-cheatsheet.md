---
title: CSS Grid 常用布局速查表
date: 2026-07-31
tags: [CSS, 前端]
description: 整理了日常开发最常用的 Grid 布局写法，从 fr 单位、模板速记到对齐系统，配合完整可抄的代码模板。
---

Grid 一出手，布局焦虑全没有。下面是平时写业务最常用的几种写法，以及藏在写法背后的关键概念。

## 先理解 fr 单位

`fr`（fraction，分数）是 Grid 独有的弹性单位，表示**剩余空间的比例分配**：

```css
.container {
  display: grid;
  grid-template-columns: 240px 1fr 2fr;
  /*         固定宽度  ① ②    */
  /* 两个 fr 列按 1:2 瓜分 240px 之外的所有剩余宽度 */
}
```

与 `flex: 1` 类似的逻辑，但更干净：`240px 1fr 2fr` 一眼可读，无需 `flex-basis` 那套。

> 对比：`1fr` 和 `auto` 不同——`auto` 按内容收缩，`1fr` 至少占据一整行剩余空间。想保持列宽不被内容撑破，用 `minmax` 兜底（见下）。

## 两栏布局（侧边栏 + 主内容）

```css
.layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  gap: 24px;
}

/* 希望侧边栏收缩、主内容占满剩余： */
.layout {
  grid-template-columns: minmax(180px, auto) 1fr;
}
```

## 响应式卡片墙（auto-fit + minmax）

```css
.cards {
  display: grid;
  gap: 16px;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
}
```

**这是本节最重要的一个写法**：每张卡片至少 280px，屏幕宽就自动增多列、窄就自动换行，零媒体查询。

### auto-fit vs auto-fill 的差别

两者都是"能放几列放几列"，差别在于**列多出来了怎么办**：

```css
/* auto-fill：即使没有内容，也留出空轨道（常用于滚动容器占位） */
grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));

/* auto-fit：空轨道被折叠，内容列拉伸填满（卡片墙用这个） */
grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
```

一句话：**卡片墙用 `auto-fit`，需要固定占位格子用 `auto-fill`**。

## 一行实现水平垂直居中

```css
.center {
  display: grid;
  place-items: center;
}
```

`place-items` 是 `align-items`（垂直）+ `justify-items`（水平）的合并速记。记住这一行，居中再也不用 `position: absolute` 那套了。

## 完整的后台布局模板（grid-template-areas）

适合整页骨架：头部、侧边栏、主区、页脚。用**命名区域**写布局，结构一目了然：

```css
.app {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 240px 1fr;
  grid-template-rows: auto 1fr auto;  /* 中间行撑满，头部页脚自适应 */
  height: 100vh;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.footer  { grid-area: footer; }
```

换布局只需改 `grid-template-areas` 的 ASCII 图：

```css
/* 手机端：侧边栏挪到底部 */
@media (max-width: 768px) {
  .app {
    grid-template-areas:
      "header"
      "main"
      "sidebar"
      "footer";
    grid-template-columns: 1fr;
  }
}
```

## 隐式网格：grid-auto-rows

不指定行数时，多出来的行是"隐式轨道"，默认 `auto`（按内容）。要控制它们（比如让所有卡片等高）：

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  grid-auto-rows: 1fr;   /* 所有自动行等高 */
}
```

`grid-auto-flow` 则控制自动排列方向：默认 `row`（逐行排），`column`（逐列排）、`dense`（允许洞补位）。

## 对齐系统：两条轴四个属性

| 属性 | 作用轴 | 作用对象 | 示例 |
| --- | --- | --- | --- |
| `justify-items` | 水平 | 单元格内内容 | `center` 让格子内水平居中 |
| `align-items` | 垂直 | 单元格内内容 | `center` 让格子内垂直居中 |
| `justify-content` | 水平 | 整个轨道 | `space-between` 轨道两端对齐 |
| `align-content` | 垂直 | 整个轨道 | `center` 轨道整体垂直居中 |

```css
/* 内容在格子里居中 */
.grid {
  justify-items: center;
  align-items: center;
  /* 等价简写：place-items: center; */
}

/* 轨道整体水平两端对齐、垂直居中（当列宽小于容器时） */
.grid {
  justify-content: space-between;
  align-content: center;
}
```

## Grid 与 Flexbox 怎么选

| 维度 | Grid | Flexbox |
| --- | --- | --- |
| 模型 | 二维（行 + 列同时控制） | 一维（单主轴） |
| 典型场景 | 整页骨架、卡片墙、表格 | 按钮组、导航栏、单行排列 |
| 对齐 | 行列都可精确控制 | 沿主轴 + 交叉轴 |
| 换行 | 由轨道定义（auto-fit 自动） | 需 `flex-wrap` + 手动设置宽度 |

**经验法则**：内容是"围着内容走"的用 Flexbox（一排按钮、导航），结构是"区域划分"的用 Grid（页面骨架、卡片墙）。两者可以嵌套混用——外层 Grid 切区域，区域内小布局用 Flex。

## 速查：一行式布局汇总

```css
/* 垂直居中 */
.center { display: grid; place-items: center; }

/* 侧边栏 + 内容 */
.side { grid-template-columns: 240px 1fr; }

/* 卡片墙（响应式） */
.cards { grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); }

/* 等高卡片 */
.cards { grid-auto-rows: 1fr; }

/* 等宽分栏 */
.cols { grid-template-columns: repeat(3, 1fr); }

/* 内容区外左右留白 */
.wrap { grid-template-columns: 1fr minmax(0, 800px) 1fr; }
```

最后一行的 `minmax(0, 800px)` 是防溢出技巧：给内容列设上限，让两侧 `1fr` 吃掉多余宽度，内容永远居中且不超宽。