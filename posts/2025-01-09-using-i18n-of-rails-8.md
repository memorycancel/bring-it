---
title: 使用 Rails 8 提供的默认国际化 I18n
layout: home
---

# 使用 Rails 8 提供的默认国际化 I18n

2025-01-09 18:00

## 1. 简介

Ruby I18n（国际化的缩写）gem 随 Ruby on Rails（从 Rails 2.2 开始）一起提供了一个易于使用且可扩展的框架，用于将的应用程序翻译成英语以外的多种自定义语言。

或者引用维基百科的话："国际化是指在设计软件应用程序时，使其能够在不进行工程改动的情况下适用于各种语言和地区。本地化是通过添加本地化组件和翻译文本，使软件适用于特定地区或语言的过程"。

国际化是一个复杂的问题。自然语言在许多方面（如复数化规则）存在差异，因此很难同时提供解决所有问题的工具。因此，Rails I18n API 专注于以下2方面：

+ 开箱即支持英语和类似(字母系)语言
+ 轻松定制和扩展其他语言的所有功能

作为该解决方案的一部分，Rails 框架中的每个静态字符串（如 Active Record 验证信息、时间和日期格式）都已国际化。Rails 应用程序的本地化意味着为这些字符串定义所需语言的翻译值。

Ruby I18n gem 分成两个部分：

+ 公共方法(API)
+ 默认后端

I18n API 最重要的方法有:

```ruby
translate # 查找翻译
localize  # 将日期和时间对象本地化为本地格式
```

它们的别名是 #t 和 #l，因此可以这样使用：

```ruby
I18n.t "store.title"
I18n.l Time.now
```

同时可以配置以下属性:

```ruby
load_path                 # 申明自定义翻译文件路径
locale                    # 获取和设置当前区域(locale)
default_locale            # 获取和设置默认区域
available_locales         # 设置应用提供翻译的区域
enforce_available_locales # 强制设置应用提供翻译的区域(true or false)
exception_handler         # 使用自定义的异常处理
backend                   # 使用自定义后端
```

## 2. 使用

接上一篇,完成了solid [queue cable cache]继承后,以此为基础集成 `i18n` :

[https://github.com/memorycancel/rails-solid/tree/i8n](https://github.com/memorycancel/rails-solid/tree/i8n)

### 2.1 配置

遵循`惯例大于配置`的理念，Rails I18n 提供了合理的默认翻译字符串。当需要不同的翻译字符串时，可以覆盖它们。
Rails 会自动将 config/locales 目录中的所有 .rb 和 .yml 文件添加到翻译加载路径。
该目录中的默认 en.yml locale 包含一对翻译字符串示例：

```ruby
en:
  hello: "Hello world"
  post: "文章"
  comment: "评论"
```

这意味着，在 :en 本地语言(locale)中，键 hello 将映射为 Hello world 字符串。Rails 中的每个字符串都是以这种方式国际化的，例如，请参阅 activemodel/lib/active_model/locale/en.yml 
文件中的 Active Model 验证信息或 activesupport/lib/active_support/locale/en.yml 
文件中的时间和日期格式。你可以使用 YAML 或标准 Ruby 哈希值来存储默认（简单）后端的翻译。

{: .note :}
i18n 库（经讨论后）对本地化键采取了一种实用的方法，只包括本地化（"语言"）部分，如 :en 、 :pl ，而不包括区域部分，如 
:"en-US" 或 :"en-GB" ，传统上，这些部分用于区分 "语言 "和 "区域设置 "或 "方言"。许多国际应用程序只使用本地的 "语言
 "元素，如 :cs 、 :th 或 :es （用于捷克语、泰语和西班牙语）。然而，在不同的语言组中，地区差异也可能很重要。例如，在 
 :"en-US" 地区，货币符号是 $，而在 :"en-GB" 地区，货币符号是 £。
 没有什么可以阻止以这种方式将区域设置和其他设置分开：只需在 :"en-GB" 字典中提供完整的 "英语 - 英国 "区域设置即可。

 翻译加载路径 ( I18n.load_path ) 是一个数组，包含将自动加载的文件路径。配置此路径可自定义翻译目录结构和文件命名方案。
可以在 `config/application.rb` 中更改默认语言以及配置翻译加载路径，具体如下：

```ruby
config.i18n.load_path += Dir[Rails.root.join("my", "locales", "*.{rb,yml}")]
config.i18n.default_locale = :zh # 中文
```
在查找任何翻译之前，必须指定加载路径。要从初始化程序而不是 config/application.rb 中更改默认本地化：

```ruby
# config/initializers/locale.rb

# Where the I18n library should search for translation files
I18n.load_path += Dir[Rails.root.join("lib", "locale", "*.{rb,yml}")]

# Permitted locales available for the application
I18n.available_locales = [:en, :zh]

# Set default locale to something other than :en
I18n.default_locale = :zh

```

{: .note :}
直接追加到 I18n.load_path 而不是应用程序配置的 I18n 不会覆盖外部 gem 的翻译。

### 2.2 管理每个请求的语言设置

国际化应用程序可能需要支持多种国家地区语言。为此，应在每次请求开始时设置本地语言，
以便在该请求的生命周期内使用所需的本地语言翻译所有字符串。
除非使用了 I18n.locale= 或 I18n.with_locale ，否则所有翻译都将使用默认的语言。

如果不在每个控制器(controller)中一致设置 I18n.locale ，它可能会泄漏到同一线程/进程服务的后续请求中。
例如，在一个 POST 请求中执行 I18n.locale = :es 会影响到以后所有未设置 locale 的控制器请求，
但仅限于该特定线程/进程。因此，你可以使用 `I18n.with_locale` 代替 `~~I18n.locale =~~` ，这样就不会出现泄漏问题。

locale 可以在 ApplicationController 中的 around_action 中设置：

```ruby
around_action :switch_locale

def switch_locale(&action)
  locale = params[:locale] || I18n.default_locale
  I18n.with_locale(locale, &action)
end
```
本示例使用 URL 查询参数设置本地化（如 http://example.com?locale=en ）来说明这一点。使用这种方法， http://localhost:3000?locale=zh 将显示中文本地化，而 http://localhost:3000?locale=en 将加载英文本地化。


### 2.3 从域名中设置本地语言

可以选择从应用程序运行的域名中设置本地语言。例如，我们希望 www.example.com 加载英语语言，而 www.example.com.cn 加载中国汉语语言。这样，顶级域名就被用于设置本地语言。这样做有几个好处：

+ 地域是 URL 的明显组成部分。
+ 人们可以直观地掌握内容将以何种语言显示。
+ 在 Rails 中实现这一功能非常简单。
+ 搜索引擎喜欢不同语言的内容存在于不同的、相互关联的域中。

可以在 ApplicationController 中这样实现该功能：

```ruby
around_action :switch_locale

def switch_locale(&action)
  locale = extract_locale_from_tld || I18n.default_locale
  I18n.with_locale(locale, &action)
end

# Get locale from top-level domain or return +nil+ if such locale is not available
# You have to put something like:
#   127.0.0.1 application.com
#   127.0.0.1 application.it
#   127.0.0.1 application.pl
# in your /etc/hosts file to try this out locally
def extract_locale_from_tld
  parsed_locale = request.host.split(".").last
  I18n.available_locales.map(&:to_s).include?(parsed_locale) ? parsed_locale : nil
end

```

我们还可以用非常类似的方法从子域中设置 locale：

```ruby
# Get locale code from request subdomain (like http://it.application.local:3000)
# You have to put something like:
#   127.0.0.1 it.application.local
# in your /etc/hosts file to try this out locally
#
# Additionally, you need to add the following configuration to your config/environments/development.rb:
#   config.hosts << 'it.application.local:3000'
def extract_locale_from_subdomain
  parsed_locale = request.subdomains.first
  I18n.available_locales.map(&:to_s).include?(parsed_locale) ? parsed_locale : nil
end
```

如果的应用程序包含一个本地语言切换菜单，就可以在其中设置如下内容：

```ruby
link_to("Deutsch", "#{APP_CONFIG[:deutsch_website_url]}#{request.env['PATH_INFO']}")
```

假设将 APP_CONFIG[:deutsch_website_url] 设置为 http://www.application.de 这样的值。
最明显的解决方案是在 URL 参数（或请求路径）中包含本地化代码。

### 2.4 通过 URL 参数设置地区语言

设置（和传递）locale 的最常用方法是将其包含在 URL 参数中，
就像我们在第一个示例中的 I18n.with_locale(params[:locale], &action) around_action 所做的那样。
在这种情况下，我们希望使用 www.example.com/books?locale=ja 或 www.example.com/ja/books 这样的 URL。

这种方法的优点与通过域名设置 locale 几乎相同：即它是 RESTful 的，并且与万维网的其他部分保持一致。不过，实现起来确实需要更多的工作。

从 params 中获取本地语言并进行相应设置并不难，难的是在每个 URL 中都包含该语言，并通过请求进行传递。
当然，在每个 URL 中都包含一个明确的选项（例如 link_to(books_url(locale: I18n.locale)) ）将会非常繁琐，而且很可能是不可能的。

Rails 的 ApplicationController#default_url_options 中包含了 "集中管理 URL 动态决策 "的基础架构，在这种情况下非常有用：它能让我们为 url_for 和依赖于它的辅助方法（通过实现/覆盖 default_url_options ）设置 "默认值"。

这样，我们就可以在 ApplicationController 中加入这样的内容：

```ruby
def default_url_options
  { locale: I18n.locale }
end
```

现在，每个依赖于 url_for 的辅助方法（例如，命名路由的辅助方法，如 root_path 或 root_url ；资源路由的辅助方法，如 books_path 或 books_url 等）都会在查询字符串中自动包含本地语言，就像下面这样： http://localhost:3001/?locale=ja .

不过，当本地语言 "挂 "在应用程序中每个 URL 的末尾时，确实会影响 URL 的可读性。此外，从架构的角度来看，locale 通常高于应用程序域的其他部分：URL 也应反映这一点。

为了是URL更美观,更RESTFUL,这可以通过上面的 "覆盖 default_url_options "策略来实现.只需用 scope 设置路由即可：

```ruby
scope "/:locale" do
  resources :posts
end
```

现在，当调用 books_path 方法时，得到 "/en/posts" （默认语言）。
然后，像 http://localhost:3001/zh/posts 这样的 URL 将加载中文汉语版本，
接下来调用 books_path 将返回 "/zh/posts" （因为语言版本发生了变化）.

### 2.5 通过用户首选项设置地域设置

有认证用户的应用程序可允许用户通过应用程序的界面设置地域偏好。
通过这种方法，用户选择的地域偏好会被持久保存在数据库中，并用于为该用户的认证请求设置地域。

```ruby
around_action :switch_locale

def switch_locale(&action)
  locale = current_user.try(:locale) || I18n.default_locale
  I18n.with_locale(locale, &action)
end
```

### 2.6 选择隐含的本地语言

如果没有为请求设置明确的本地语言（例如通过上述方法之一），应用程序应尝试推断出所需的本地语言。

### 2.7 从语言头推断地域

`Accept-Language` HTTP标头表示请求响应的首选语言。
浏览器会根据用户的语言偏好设置来设置该标头值，因此它是推断地域的首选。
使用 Accept-Language 标头的一个简单实现方法是

```ruby
def switch_locale(&action)
  logger.debug "* Accept-Language: #{request.env['HTTP_ACCEPT_LANGUAGE']}"
  locale = extract_locale_from_accept_language_header
  logger.debug "* Locale set to '#{locale}'"
  I18n.with_locale(locale, &action)
end

private
  def extract_locale_from_accept_language_header
    request.env["HTTP_ACCEPT_LANGUAGE"].scan(/^[a-z]{2}/).first
  end
```

在实践中，需要更强大的代码才能可靠地做到这一点。Iain Hecker 的 http_accept_language 库或 Ryan Tomayko 的 locale Rack 中间件为这一问题提供了解决方案。

### 2.8 从 IP 地理位置推断位置

发出请求的客户端的 IP 地址可以用来推断客户端所在的地区，从而推断出他们的地域。可以使用 GeoLite2 Country 等服务或 geocoder 等gem来实现这种方法。

总的来说，这种方法远不如使用语言头可靠，不推荐用于大多数网络应用程序。

### 2.9 从会话或 Cookie 中存储位置信息

{: .important :}
您可能想将所选的语言保存在会话或 cookie 中。但千万不要这样做。地域应该是透明的，是 URL 
的一部分。这样你就不会打破人们对网络本身的基本假设：如果你向朋友发送一个 
URL，他们`看到的页面和内容应该和你一样`。用一个花哨的词来形容就是 "RESTful"。有关 RESTful 方法的更多信息，请参阅 
Stefan Tilkov 的文章。有时，这一规则也有例外，下文将对此进行讨论。


## 3. 国际化和本地化

应用程序初始化了 I18n 支持，并告诉它要使用哪种语言，以及如何在请求之间保存语言。
接下来，我们需要通过抽象每个特定于本地的"字符串"来实现应用程序的国际化。
最后，我们需要通过为这些抽象提供必要的翻译来实现本地化。

{: .note :}
也就是说,国际化是给将来需要翻译的字段留好坑位(抽象字段).并在相关位置`t()`
本地化就是提供在locales.yml翻译文件的过程.

### 3.1 抽象本地化代码

`/home/memorycancel/git/rails-solid/app/views/posts/index.html.erb`:

```ruby
<h1><%= t :posts %></h1>
```

`/home/memorycancel/git/rails-solid/app/views/comments/_comments.html.erb`:

```ruby
<h2><%= t :comments %></h2>
```

### 3.2 为国际化字符串提供翻译

```ruby
# config/locales/en.yml
en:
  posts: Posts
  comments: Comments

# config/locales/zh.yml
zh:
  posts: 文章
  comments: 评论
```

{: important :}
添加新的本地化文件时，需要重新启动服务器。

可以使用 YAML ( .yml ) 或纯 Ruby ( .rb ) 文件在 SimpleStore 中存储翻译。
YAML 是 Rails 开发人员的首选。不过，它有一个很大的缺点。YAML 对空白和特殊字符非常敏感，
因此应用程序可能无法正确加载词典。Ruby 文件会在第一次请求时让应用程序崩溃，
因此您可以很容易地找到问题所在。(如果您在使用 YAML 词典时遇到任何 "奇怪的问题"，
请尝试将词典的相关部分放入 Ruby 文件中）。

如果翻译存储在 YAML 文件中，某些键必须转义。它们是:

```ruby
en:
  success:
    'true':  'True!'
    'on':    'On!'
    'false': 'False!'
  failure:
    true:    'True!'
    off:     'Off!'
    false:   'False!'
```

```ruby
I18n.t "success.true"  # => 'True!'
I18n.t "success.on"    # => 'On!'
I18n.t "success.false" # => 'False!'
I18n.t "failure.false" # => Translation Missing
I18n.t "failure.off"   # => Translation Missing
I18n.t "failure.true"  # => Translation Missing
```

### 3.3 向翻译传递变量

成功实现应用程序国际化的一个关键因素是在抽象本地化代码时避免对语法规则做出错误的假设。
为了创建适当的抽象，I18n gem 提供了一种称为变量插值的功能，允许您在翻译定义中使用变量，
并将这些变量的值传递给翻译方法。

```ruby
<!-- app/views/products/show.html.erb -->
<%= t('product_price', price: @product.price) %>

# config/locales/en.yml
en:
  product_price: "$%{price}"

# config/locales/zh.yml
zh:
  product_price: "%{price*7} 元"
```

### 3.4 添加日期/时间格式

要本地化时间格式，可以将时间对象传入 I18n.l 或（最好）使用 Rails 的 #l 助手。你可以通过 :format 选项选择一种格式，默认情况下使用 :default 格式。

```ruby
# /home/memorycancel/git/rails-solid/app/views/comments/_comment.html.erb
<div id="<%= dom_id(comment) %>">
  <%= comment.content %> -
  <%= l comment.updated_at, format: :short %>
</div>
```

```ruby
# config/locales/en.yml
en:
  posts: Posts
  comments: Comments
  time:
    formats:
      short: "%Y%M%D%H%M%S"

# config/locales/zh.yml
zh:
  posts: 文章
  comments: 评论
  time:
    formats:
      short: "%Y年%M月%D日: %H%M%S"
```

### 3.5 字母系语言语法转换规则

Rails 允许您为英语以外的本地语言定义词形变化规则（如单复数化和复数化规则）。
在 config/initializers/inflections.rb 中，您可以为多个本地语言定义这些规则。
初始化器包含一个默认示例，用于指定英语的附加规则；对于其他本地语言，请按照您认为合适的格式进行。

### 3.6 本地化文件组织

将应用程序所有部分的翻译都放在每个本地语言的一个文件中可能难以管理。
可以将模型和模型属性名称与视图内的文本分开，并将所有这些与 "默认值"（如日期和时间格式）分开。i18n 库的其他存储空间可以提供不同的分离方式。

```ruby
# config/locales
|-defaults
|---es.yml
|---en.yml
|-views
|---defaults
|-----es.yml
|-----en.yml
|---posts
|-----es.yml
|-----en.yml
|---comments
|-----es.yml
|-----en.yml
```
{: .important :}
Rails 的默认本地化加载机制并不像我们这里这样在嵌套字典中加载本地化文件。因此，我们必须明确告诉 Rails 进一步查找，才能实现这一功能

```ruby
# config/application.rb
config.i18n.load_path += Dir[Rails.root.join("config", "locales", "**", "*.{rb,yml}")]
```

### I18n API

详尽的API细节参考: [https://guides.rubyonrails.org/i18n.html#overview-of-the-i18n-api-features](https://guides.rubyonrails.org/i18n.html#overview-of-the-i18n-api-features)