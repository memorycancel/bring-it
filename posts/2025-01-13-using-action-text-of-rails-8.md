---
title: 使用 Rails 8 提供的默认富文本模块
layout: home
---

# 使用 Rails 8 提供的默认富文本模块

2025-01-13 23:00

## 1 什么是 Action Text？

`Action Text` 可帮助处理和显示富文本内容。富文本内容是包含格式元素（如粗体、斜体、颜色和超链接）的文本，它提供了超越纯文本的视觉增强和结构化显示。它允许我们创建富文本内容，将其存储在表格中，然后将其附加到我们的任何模型中。

Action Text 包含一个名为 Trix（[https://en.wikipedia.org/wiki/WYSIWYG](https://en.wikipedia.org/wiki/WYSIWYG） 的所见即所得编辑器，它用于网络应用程序，为用户创建和编辑富文本内容提供友好的用户界面。
它可以处理一切内容，包括提供丰富的功能，如文本格式、添加链接或引语、嵌入图片等。有关示例，
请参见 Trix 编辑器网站[https://trix-editor.org/](https://trix-editor.org/。

{: .note :}
大多数所见即所得编辑器都是 HTML 的 contenteditable 和 execCommand API 的封装器。这些 API 由微软设计，用于支持在 Internet Explorer 5.5 中实时编辑网页。它们最终被其他浏览器反向设计和复制。由于所见即所得的 HTML 编辑器范围巨大，每个浏览器的实现都有自己的错误和怪癖。因此，JavaScript 开发人员往往只能自己解决不一致的问题。Trix 将 contenteditable 视为 I/O 设备，从而避免了这些不一致性：当输入到达编辑器时，Trix 会将输入转换为内部文档模型上的编辑操作，然后将文档重新渲染回编辑器。这样，Trix 就能完全控制每次按键后发生的事情，避免了使用 execCommand 和随之而来的不一致性。

Trix 编辑器生成的富文本内容保存在自己的 RichText 模型中，该模型可与应用程序中任何现有的 Active Record 
模型关联。此外，任何内嵌图片（或其他附件）都可以使用 Active Storage（作为依赖项添加）自动存储，并与 RichText 
模型关联。在渲染内容时，Action Text 会首先对内容进行`消毒处理`，以便安全地直接嵌入到页面的 HTML 中。

{: .important :}
使用默认的 Action Text 可以获得开相机用的上传文件（副文本存储）安全。

## 2 安装

```shell
bin/rails action_text:install
```
它将执行以下操作：

+ 安装 trix 和 @rails/actiontext 的 JavaScript 包，并将它们添加到 application.js 中。
+ 通过 Active Storage 添加 image_processing gem，用于分析和转换嵌入式图像和其他附件。有关详细信息，请参阅 [Active Storage](https://guides.rubyonrails.org/active_storage_overview.html) 概述指南。
+ 添加迁移功能，以创建存储富文本内容和附件的`数据库表`： action_text_rich_texts , active_storage_blobs , active_storage_attachments , active_storage_variant_records .
+ 创建包含所有 Trix 风格和重载的 actiontext.css 。
+ 添加默认视图部分 `_content.html` 和`_blob.html` ，以分别呈现 Action Text 内容和 Active Storage 附件（又名 blob）。

之后，执行迁移会将新的 action_text_* 和 active_storage_* 表添加到应用程序中：

```shell
bin/rails db:migrate
```
当 "动作文本 "安装程序创建 action_text_rich_texts 表时，它会使用多态关系（可以`belongs_to`），
以便多个模型可以添加富文本属性。这是通过 record_type 和 record_id 列实现的，这两列分别存储模型的 ClassName 和记录的 ID。

```ruby
t.references :record, null: false, polymorphic: true, index: false, type: :uuid
```

## 3 创建富文本内容

RichText 记录将 Trix 编辑器生成的内容保存在序列化的 body 属性中。它还包含嵌入文件的所有引用，这些引用使用 Active Storage 存储。然后，将此记录与希望拥有富文本内容的 Active 记录模型关联起来。通过在希望添加富文本内容的模型中放置 has_rich_text 类方法，就可以建立关联。

```ruby
class Post < ApplicationRecord
  has_many :comments
  has_rich_text :content # 添加此行
end
```

将 has_rich_text 类方法添加到模型后，就可以更新视图，以便使用该字段的富文本编辑器（Trix）。在表单字段中使用 rich_textarea 。

```ruby
<%# app/views/posts/_form.html.erb %>
<%= form_with(model: post) do |form| %>
  <% if post.errors.any? %>
    <div style="color: red">
      <h2><%= pluralize(post.errors.count, "error") %> prohibited this post from being saved:</h2>

      <ul>
        <% post.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :title, style: "display: block" %>
    <%= form.text_field :title %>
  </div>

  <div>
    <%= form.label :body, style: "display: block" %>
    <%= form.rich_textarea :body %> <%# 修改此处，将 textarea 修改为 rich_textarea %>
  </div>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

由于 Action Text 依赖于多态关联，而多态关联又涉及在数据库中存储类名，因此保持数据与 Ruby 
代码中使用的类名同步至关重要。这种同步对于保持存储数据与代码库中类引用之间的一致性至关重要。

如果是生产环境，还需要预编译：

```
export RAILS_ENV=prodction
rails assets:precompile
```

## 4 渲染富文本内容

ActionText::RichText 实例可以直接嵌入到页面中，因为它们已经对其内容进行了安全渲染处理。您可以按如下方式显示内容：

```
<%= @post.body %>
```

ActionText::RichText#to_s 可以安全地将 RichText 转换为 HTML 字符串。
另一方面， ActionText::RichText#to_plain_text 返回的字符串不是 HTML 安全字符串，
不应在浏览器中呈现。你可以在 [ActionText::RichText 文档](https://api.rubyonrails.org/v8.0.1/classes/ActionText/RichText.html)中了解更多有关 Action Text 净化过程的信息。

## 5 自定义富文本内容编辑器（Trix）

默认情况下，Action Text 将在带有 .trix-content 类的元素内呈现富文本内容。
该类在 `app/views/layouts/action_text/contents/_content.html.erb` 中设置。
具有该类的元素将由 trix 样式表进行样式设置。

如果你想更新任何 trix 样式，可以在 app/assets/stylesheets/actiontext.css 中添加你的自定义样式，
其中包括 Trix 的全套样式和 Action Text 所需的覆盖样式。

要自定义富文本内容周围呈现的 HTML 容器元素，请编辑安装程序创建的 `app/views/layouts/action_text/contents/_content.html.erb` 布局文件：

```ruby
<%# app/views/layouts/action_text/contents/_content.html.erb %>
<div class="trix-content">
  <%= yield %>
</div>
```
要自定义嵌入图片和其他附件（称为 Blob）的 HTML 呈现，请编辑安装程序创建的 `app/views/active_storage/blobs/_blob.html.erb 模板`：

```ruby
<figure class="attachment attachment--<%= blob.representable? ? "preview" : "file" %> attachment--<%= blob.filename.extension %>">
  <% if blob.representable? %>
    <%= image_tag blob.representation(resize_to_limit: local_assigns[:in_gallery] ? [ 800, 600 ] : [ 1024, 768 ]) %>
  <% end %>

  <figcaption class="attachment__caption">
    <% if caption = blob.try(:caption) %>
      <%= caption %>
    <% else %>
      <span class="attachment__name"><%= blob.filename %></span>
      <span class="attachment__size"><%= number_to_human_size blob.byte_size %></span>
    <% end %>
  </figcaption>
</figure>
```

## 6 附件

目前，Action Text 支持通过 Active Storage 上传的附件以及与 Signed GlobalID 关联的附件。

在富文本编辑器中上传图片时，会使用 Action Text，而 Action Text 又会使用 Active Storage。
不过，Active Storage 有一些 Rails 不提供的依赖库。要使用内置预览器，必须安装这些库。
这些库中有些（但不是全部）是必需的，它们取决于编辑器中预期的上传类型。

{: .important :}
用户在使用 Action Text 和 Active Storage 时遇到的一个常见错误是，图片无法在编辑器中正确呈现。这通常是由于 `libvips` 依赖项未安装所致。

```shell
snap install libvips
```

关于更多魔改编辑器附件参考：[https://guides.rubyonrails.org/action_text_overview.html#attachment-direct-upload-javascript-events](https://guides.rubyonrails.org/action_text_overview.html#attachment-direct-upload-javascript-events),
例如：可以自己添加一个视频附件播放器，或者PDF预览等。

## 7 避免 N+1 查询

如果要预载依赖的 ActionText::RichText 模型，假设富文本字段的名称是 body ，则可以使用命名作用域：

```ruby
Post.all.with_rich_text_content # Preload the body without attachments.
Post.all.with_rich_text_content_and_embeds # Preload both body and attachments.
```
