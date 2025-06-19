---
title: Rails 8 默认 JavaScript 框架 Stimulus 高级用法
layout: home
---

# Rails 8 默认 JavaScript 框架 Stimulus 高级用法

## 1. 访问浏览器系统 API 实现剪贴板

app/views/posts/_post.html.erb
```html
    ...
    <div data-controller="clipboard">
      <input data-clipboard-target="source" type="text" value="<%= post.title %>" readonly>
      <button data-action="clipboard#copy">Copy to Clipboard</button>
    </div>
```

{: .important :}
遵循“渐进式增强”理念，首先实现 HTML，然后添加 Stimulus 三要素：controller，action，target。
data-controller 属性与 js 文件名对应，data-action 属性与 js 里的方法名对应，target 为需要进行操作的目标。

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "source" ]
  copy(event) {
    navigator.clipboard.writeText(this.sourceTarget.value)
    alert('success')
  }
}
```
`navigator.clipboard.writeText` 是现代浏览器标准 js 剪贴板 [API](https://www.w3.org/TR/clipboard-apis/)。


每个 elements 都有默认的 events，所以上面`data-action="click->clipboard#copy"`实际上省略了 `click->`。

| Element                | Default Event  |
|:--------------------|:------------------|
|a|  click|
|button|  click|
|details| toggle|
|form|    submit|
|input|   input|
|input type=submit|   click|
|select|  change|
|textarea|    input|

## 2. 通过状态管理实现幻灯片

大多数当代框架都鼓励始终将状态保存在 JavaScript 中。它们将 DOM 视为只可写入的渲染目标，通过客户端模板从服务器上消费 JSON 来调节。

Stimulus 采用了不同的方法。Stimulus 应用程序的状态以属性的形式存在于 DOM 中；而控制器本身在很大程度上是无状态的。通过这种方法，我们可以在任何地方（初始文档、Ajax 请求、Turbo 访问，甚至是另一个 JavaScript 库）处理修改 HTML，并且无需任何显式初始化步骤，相关控制器就会自动启动。

app/views/posts/index.html.erb
```html
<div data-controller="slideshow" data-slideshow-index-value="1">
  <button data-action="slideshow#previous"> ← </button>
  <button data-action="slideshow#next"> → </button>

  <div data-slideshow-target="slide">🐵</div>
  <div data-slideshow-target="slide">🙈</div>
  <div data-slideshow-target="slide">🙉</div>
  <div data-slideshow-target="slide">🙊</div>
</div>
```
+ controller: 一般为最外层 div DOM
+ action: 有触发事件（例如：click）的 div DOM
+ target: 需要被修改的 div DOM
+ value: 一般放在需要调用值的外层 div

app/javascript/controllers/slideshow_controller.js
```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "slide" ]
  static values = { index: Number }

  next() {
    if (this.indexValue < 3) {
      this.indexValue++
    }
    else {
      this.indexValue -= 3;
    }
  }

  previous() {
    if (this.indexValue > 0)
    {
      this.indexValue--
    }
    else {
      this.indexValue += 3
    }
  }

  indexValueChanged() {
    this.showCurrentSlide()
  }

  showCurrentSlide() {
    this.slideTargets.forEach((element, index) => {
      element.hidden = index !== this.indexValue
    })
  }
}
```
### 2.1 生命周期

控制器定义了一个方法 showCurrentSlide() ，该方法在每个幻灯片目标上循环，如果索引匹配，则切换 hidden 属性。
显示第一张幻灯片来初始化控制器，而 next() 和 previous() 操作方法则前进和后退当前幻灯片。
initialize() 方法有什么作用？它与我们之前使用过的 connect() 方法有何不同？

| Method                | Invoked by Stimulus...  |
|:--------------------|:------------------|
|initialize() |  一旦控制器首次实例化|
|connect()|  控制器连接到 DOM 的任何时候|
|disconnect() | 控制器与 DOM 断开连接时|


这些都是 Stimulus 生命周期回调方法，当控制器进入或离开 DOM 时，它们可以用来设置或删除相关状态。
这里我们使用 initialize() 设置幻灯片的初始状态。

### 2.2 使用 Stimulus 的值 values

{: .note :}
可以理解为`values`为Stimulus中另外一个有同的关键字，类似`targets`。

Stimulus 控制器支持自动映射到数据属性的类型值属性。当我们在控制器类的顶部添加一个值定义时：
```javascript
  static values = { index: Number }
```
Stimulus 将创建一个与 data-slideshow-index-value 属性相关联的 this.indexValue 控制器属性，并为我们处理数值转换。
在我们的 HTML 中添加相关的数据属性：

```html
<div data-controller="slideshow" data-slideshow-index-value="1">
```

这样我们就能在 js 中使用 `this.indexValue` 操纵数据。假如我们需要加入别的值：

```javascript
  static values = { index: Number, other: Number }
```

### 2.3 设置默认值

还可以在静态定义中设置默认值：

```javascript
  static values = { index: { type: Number, default: 2 } }
```

如果控制器元素上没有定义 data-slideshow-index-value 属性，索引将从 2 开始。如果你有其他值，你可以混合并匹配哪些需要默认值，哪些不需要：

```javascript
  static values = { index: Number, effect: { type: String, default: "kenburns" } }
```

源码：[https://github.com/memorycancel/rails-8-demo/compare/stimulus](https://github.com/memorycancel/rails-8-demo/compare/stimulus)
