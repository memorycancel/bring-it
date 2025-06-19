---
title: 使用 Rails 8 的 propshaft 集成 bootstrap 5.3.6
layout: home
---

# 使用 Rails 8 的 propshaft 集成 bootstrap 5.3.6

## 0. 前言

得益于当今硬件和网络的升级，Rails 8 替换了 Sprockets 使用 propshaft 回归了网站 APP 开发传统，
使用更简单的配置和更清晰的 assets 路径管理。繁杂的`js/css`build打包压缩让人头秃，而简约的 assets 配置是“码农”的福音。

## 1. 目的

如何手工集成 bootstrap 到 rails 8 框架呢？
rails 8 默认使用 propshaft，需要集成的 bootstrap 框架包含 2 大部分：

+ bootstrap.css
+ bootstrap.js (popperjs)

## 2. 步骤

### 2.1 css

访问 `https://getbootstrap.com/` 找到 CDN 链接，下载到项目的 `app/assets/stylesheets/bootstrap.min.css` 路径。

propshaft 会自动加载 `app/assets/stylesheets/` 下的所有 css 文件。

### 2.2 js

集成 js 使用 propshaft 推荐的 `https://github.com/rails/importmap-rails`。

项目路径下执行：
```shell
bin/importmap pin bootstrap
# 如果需要删除可以执行 unpin
```
会自动下载需要的js到 `vendor/javascript/bootstrap.js` 与 `config/importmap.rb` 新增的配置对应。

{: .note :}
执行 bin/importmap pin bootstrap 后，popperjs 有 bug，解决办法参考：https://github.com/rails/importmap-rails/issues/65#issuecomment-2049989937

```shell
bin/importmap pin @popperjs/core@2.11.8/+esm  --from jsdelivr
mv vendor/javascript/@popperjs--core--+esm.js vendor/javascript/stupid-popper-lib-2024.js

# config/importmap.rb
pin "@popperjs/core", to: "stupid-popper-lib-2024.js"
```

最后，别忘了在 `app/javascript/application.js` 引入：

```javascript
...
import * as bootstrap from "bootstrap";
```

至此，bootstrap 已经成功集成到 rails 8，尽情享用 `https://getbootstrap.com/docs/5.3/components/accordion/` 里的组件吧。

完整代码参考：

+ [https://github.com/memorycancel/rails-8-demo/commit/c1d36126d66dda09e3c8ee6f6147301df7c37d38](https://github.com/memorycancel/rails-8-demo/commit/c1d36126d66dda09e3c8ee6f6147301df7c37d38)

## 3. 更多

那么如何自定义css和js？

### 3.1 css

按照命名喜好，在 `app/assets/stylesheets/` 添加例如：
+ `app/assets/stylesheets/post_index.css`
+ `app/assets/stylesheets/post_form.css`

参考：

+ [https://guides.rubyonrails.org/getting_started.html#adding-css-javascript](https://guides.rubyonrails.org/getting_started.html#adding-css-javascript)

### 3.2 模块 js

如果想从 app/javascript/src 或 app/javascript 的其他子文件夹（如 channels ）导入本地 js 模块文件，必须将它们固定下来才能导入。可以使用 pin_all_from 选取特定文件夹中的所有文件，这样就不必 pin 逐个导入每个模块了。
```ruby
# config/importmap.rb
pin_all_from 'app/javascript/src', under: 'src', to: 'src'
```

只有更改目标逻辑导入名称时才需要 :to 参数。如果放弃 :to 选项，则必须将 :under 选项直接放在第一个参数之后。

从 app/javascript/src/example_function.js 导入函数：
```javascript
// app/javascript/application.js
import { ExampleFunction } from 'src/example_function'
```

{: .note :}
Sprockets 曾经使用逻辑相对路径提供它无法从 app/javascripts 文件夹中找到的资产（尽管没有文件名摘要），这意味着无需`pin`本地文件。Propshaft 没有这种回退机制，因此使用 Propshaft 时必须`pin`本地模块。

参考：
+ [https://github.com/rails/importmap-rails?tab=readme-ov-file#local-modules](https://github.com/rails/importmap-rails?tab=readme-ov-file#local-modules)

### 3.3 页面 js

需要自定义 JavaScript 为页面添加功能需要使用 [`Stimulus`](https://stimulus.hotwired.dev/)框架。
