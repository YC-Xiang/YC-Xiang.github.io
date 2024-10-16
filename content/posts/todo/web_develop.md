---
title: HTML, CSS, JS
date: 2024-06-29 09:57:28
tags:
  - Web Develop
categories:
  - Notes
draft: true
---

# HTML

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Title!</title>
  </head>

  <body>
    <h1>Heading!</h1> <!-- 标题, h1,h2,h3...-->
    <p>Paragraph!</p>
  </body>

<!-- inserting links -->
  <a href="https://www.google.com">Link to Google</a>

<!-- inserting images-->
  <img src="https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/RISC-V%E4%B8%AD%E6%96%87%E6%89%8B%E5%86%8C/10.5.png"></img>
<!-- or -->
  <img src="https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/RISC-V%E4%B8%AD%E6%96%87%E6%89%8B%E5%86%8C/10.5.png"/>

<!-- unordered Lists -->
  <ul>
    <li>Item 1</li>
    <li>Item 2</li>
  </ul>
<!-- ordered Lists -->
  <ol>
    <li>Item 1</li>
    <li>Item 2</li>
  </ol>

<!-- Block section-->
  <div id="div-0">
    <h2>Div 1!</h2>
    <p>Paragraph</p>
  </div>
<!-- Inline section-->
 <span id="span-0">This text is inside a span element.</span>
 <span id="span-1" span>This text is inside another span element.</span>

</html>
```

# CSS

HTML 中一种以 class 定义

```html
<div class="info">Info</div>
```

那么 CSS 中对该 class 的修改以.xxx 开始

```CSS
.info {
  color: red;
  font-family: Arial;
  font-size: 24pt;
}
```

如果 HTML 中以 id 定义，css 中以#开头。

```html
<div id="unique">Info</div>
```

```CSS
#unique {
  color: red;
  font-family: Arial;
  font-size: 24pt;
}
```

注意两者的区别：

- id: An element can have only one ID. `<div id="element-id">`
- class: An element can have multiple classes. `<div class="class1 class2 class3">`

优先级排序：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240629103026.png)

例如如果 css 中对某一元素又指定了 id 属性和 class 属性，那么 id 属性优先级更高。

# Combing HTML and CSS

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Title!</title>
    <!-- 指定css文件 -->
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <!-- 定义class，这边需要在css文件中设置格式 -->
    <h1 class="h1-class">Heading!</h1>
    <p class="p-class">Paragraph!</p>
  </body>
</html>
```

```css
.my-class {
  color: red;
  font-family: Arial;
  font-size: 24pt;
}
```

# Workshop0

CSS 中直接以 HTML 中的标签设置属性：

```css
body {
  font-family: "Open Sans", sans-serif;
}
```

对应 html 中 body 包含的所有内容。

```html
<body></body>
```

## import font

## add a navbar

css 中定义变量，在`:root`中以`--`命名:

```css
:root {
  --primary: #396dff;
  --grey: #f7f7f7;
  --white: #fff;
}
```

使用变量:

```css
.navTitle {
  color: var(--primary);
}
```

navbar: `<nav></nav>`

## Remove the margin

每个元素外面都有 margin，border，padding 三种元素，chrome 中默认的 margin 是 8.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240630140432.png)

```css
.navTitle {
  color: var(--white);
  font-size: 20px;
  /* Below cancels out some of the default styling of h1 */
  margin: 0;
  border: 10px solid black;
  padding: 10px 10px 10px 10px; /*up right down ledt*/
  font-weight: normal;
  weight: 100px;
  height: 100px;
}
```

在 chrome 中对网页右键 inspect，点击这个选择按钮，再选择网页中的元素可以查看。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240630144344.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240630144312.png)

## Border radius

`border-radius`属性可以倒圆角。

`border-raduis: 50%`可以把图形改变成圆形。

```css
.avatar {
  max-width: 100%;
  border-radius: var(--m);
}border-radius
```

## Horizontal Format

使用`flex`。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240630152920.png)

## Box Sizing

`flex-basis`(默认大小)和`flex-grow`(增长大小)

```css
.subContainer {
  flex-grow: 1;
  flex-basis: 0;
}
```

# JavaScript
