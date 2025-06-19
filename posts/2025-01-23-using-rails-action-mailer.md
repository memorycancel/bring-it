---
title: 使用 Rails 8 邮件发送器 Action Mailer
layout: home
---

# 使用 Rails 8 邮件发送器 Action Mailer

2025-01-23 17:00

## 1 什么是 Action Mailer？

Action Mailer 允许从 Rails 应用程序向外部发送电子邮件。它是 Rails 框架中与电子邮件相关的两个组件之一。另一个是 Action Mailbox，用于接收电子邮件。

Action Mailer 使用类（称为 "邮件发送器"）和视图来创建和配置要发送的电子邮件。邮件发送器是继承自 [ActionMailer::Base](https://api.rubyonrails.org/v8.0.1/classes/ActionMailer/Base.html) 类。邮件发送器类与控制器 controller 类类似。两者都有：

+ 可在视图 views 中访问实例变量
+ 使用布局 layout 和局部 partial 的功能。
+ 访问 params hash 的功能。
+ Actions 和 相关 views 的相关对应逻辑

## 2 创建邮件发送器和视图

以下是如何使用 Action Mailer 发送电子邮件步骤的详情。

### 2.1 生成邮件程序

首先，使用 "Mailer"生成器创建 Mailer 相关类：

```shell
$ bin/rails generate mailer User
create  app/mailers/user_mailer.rb
invoke  erb
create    app/views/user_mailer
invoke  test_unit
create    test/mailers/user_mailer_test.rb
create    test/mailers/previews/user_mailer_preview.rb
```

与下面的 UserMailer 一样，所有生成的 Mailer 类都继承自 ApplicationMailer ：

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
end
```
ApplicationMailer 类继承自 ActionMailer::Base ，可用于定义所有邮件发送器的共同属性：

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "from@example.com"
  layout "mailer"
end
```
如果不想使用生成器，也可以手动添加文件到 app/mailers 目录。确保类继承自 ApplicationMailer ：

```ruby
# app/mailers/custom_mailer.rb
class CustomMailer < ApplicationMailer
end
```
### 2.2 编辑邮件发送器

app/mailers/user_mailer.rb 中的 UserMailer 最初没有任何方法。
因此，接下来要为邮件发送器添加方法（又称action），以发送特定邮件。

邮件发送器的方法称为 "操作" actions，它们使用视图来组织内容，与控制器类似。控制器会生成 HTML 内容发送给客户端，而Action Mailer 则会创建要通过电子邮箱发送的信息。

在 UserMailer 中添加一个名为 welcome_email 的方法，向用户注册的电子邮件地址发送电子邮件：

```ruby
class UserMailer < ApplicationMailer
  default from: "notifications@example.com"

  def welcome_email
    @user = params[:user]
    @url  = "http://example.com/login"
    mail(to: @user.email, subject: "Welcome to My Awesome Site")
  end
end
```

{: .note :}
邮件发送器中的方法名称不必以 `_email` 结尾。以上只是一个示例。

下面是对上述邮件发送器相关方法的简要解释：

+ default 方法为该邮件程序发送的所有邮件设置默认值。在本例中用它为该类中的所有邮件设置 :from 头值。这可以根据每封邮件进行重写。
+ mail 方法创建实际的电子邮件信息。我们用它来指定每封邮件的 :to 和 :subject 标题的值。

还有一个 headers 方法（上面未使用），用于通过哈希值或调用 headers[:field_name] = 'value' 来指定邮件头。

在使用生成器时，还可以像这样直接指定 action：
```shell
$ bin/rails generate mailer User welcome_email
```
上述方法将生成带有空 welcome_email action 的 UserMailer 。
还可以用一个邮件发送器类发送多封邮件。这样可以方便地将相关邮件分组。
例如，上面的 UserMailer 除了 welcome_email 之外，还可以有 goodbye_email （以及相应的视图）。

### 2.3 创建邮件视图

对于 welcome_email action，需要在 app/views/user_mailer/ 目录下名为 welcome_email.html.erb 的文件中创建一个匹配的视图。下面是可用于欢迎电子邮件的 HTML 模板示例：

```ruby
<h1>Welcome to example.com, <%= @user.name %></h1>
<p>
  You have successfully signed up to example.com,
  your username is: <%= @user.login %>.<br>
</p>
<p>
  To login to the site, just follow this link: <%= link_to 'login`, login_url %>.
</p>
<p>Thanks for joining and have a great day!</p>
```
{: .note :}
以上是 <body> 标签的内容。它将嵌入包含 <html> 标签的默认邮件布局 layout 中。

也可以创建上述电子邮件的文本版本，并将其存储在 app/views/user_mailer/ 目录中的 welcome_email.text.erb 中（注意 .text.erb 扩展名与 html.erb 相对）。发送这两种格式的邮件被认为是最佳做法，因为在出现 HTML 渲染问题时，文本版本可以作为可靠的备用格式。下面是一封文本电子邮件的示例：

```ruby
Welcome to example.com, <%= @user.name %>
===============================================

You have successfully signed up to example.com,
your username is: <%= @user.login %>.

To login to the site, just follow this link: <%= @url %>.

Thanks for joining and have a great day!
```

{: .note :}
个人认为纯文本格式是极具 Unix 风格美感的。

请注意，在 HTML 和文本电子邮件模板中都可以使用实例变量 @user 和 @url 。
现在，当调用 mail 方法时，Action Mailer 将检测两个模板（文本和 HTML），并自动生成一封 multipart/alternative 电子邮件。

### 2.4 调用邮件发送器

设置好邮件发送器类和视图后，下一步就是实际调用邮件发送器方法来渲染电子邮件视图（即发送电子邮件）。
邮件发送器可以看作是渲染视图的另一种方式。控制器操作会呈现视图，并通过 HTTP 协议发送。而邮件发送器会呈现视图，并通过电子邮件协议发送。
因为已经使用 auth 创建了User，接下来，编辑 SessionController 中的 create 操作，以便在用户登录时发送欢迎电子邮件。为此，在成功登录后立即调用 UserMailer.with(user: @user).welcome_email 。

{: .note :}
使用 deliver_later 来排队发送电子邮件。这样，控制器操作将继续进行，而无需等待电子邮件发送代码运行。 deliver_later 方法由 Active Job 支持。

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_session_url, alert: "Try again later." }

  def new
  end

  def create
    if user = User.authenticate_by(params.permit(:email_address, :password))
      start_new_session_for user
      # Tell the UserMailer to send a welcome email after save
      UserMailer.with(user: @user).welcome_email.deliver_later
      redirect_to after_authentication_url
    else
      redirect_to new_session_path, alert: "Try another email address or password."
    end
  end

  def destroy
    terminate_session
    redirect_to new_session_path
  end
end
```
传递给 with 的任何键值对都会成为 Mailer action 的 params 。例如， with(user: @user, account: @user.account) 使 params[:user] 和 params[:account] 在 Mailer action 中可用。

设置好上述邮件发送器、视图和控制器后，如果一个新的 User 登录 ，就可以检查日志，查看发送的欢迎电子邮件。日志文件将显示发送的文本和 HTML 版本，就像这样：

```text
[ActiveJob] [ActionMailer::MailDeliveryJob] [ec4b3786-b9fc-4b5e-8153-9153095e1cbf] Delivered mail 6661f55087e34_1380c7eb86934d@Bhumis-MacBook-Pro.local.mail (19.9ms)
[ActiveJob] [ActionMailer::MailDeliveryJob] [ec4b3786-b9fc-4b5e-8153-9153095e1cbf] Date: Thu, 06 Jun 2024 12:43:44 -0500
From: notifications@example.com
To: test@gmail.com
Message-ID: <6661f55087e34_1380c7eb86934d@Bhumis-MacBook-Pro.local.mail>
Subject: Welcome to My Awesome Site
Mime-Version: 1.0
Content-Type: multipart/alternative;
 boundary="--==_mimepart_6661f55086194_1380c7eb869259";
 charset=UTF-8
Content-Transfer-Encoding: 7bit

----==_mimepart_6661f55086194_1380c7eb869259
Content-Type: text/plain;
...
----==_mimepart_6661f55086194_1380c7eb869259
Content-Type: text/html;
...
```
也可以从 Rails 控制台调用邮件发送器并发送邮件，在设置好控制器动作之前，这也许是个有用的测试。
下面将发送与上面相同的 welcome_email ：

```ruby
user = User.first
UserMailer.with(user: user).welcome_email.deliver_later
```
如果想立即发送邮件，可以调用 deliver_now ：

```ruby
class SendWeeklySummary
  def run
    User.find_each do |user|
      UserMailer.with(user: user).weekly_summary.deliver_now
    end
  end
end
```
UserMailer 的 weekly_summary 方法会返回一个 ActionMailer::MessageDelivery 对象，该对象拥有 deliver_now 或 deliver_later 方法，可以现在或稍后发送自己。 ActionMailer::MessageDelivery 对象是 Mail::Message 对象的一个封装。 如果要检查、更改 Mail::Message 对象或对其进行其他操作，可以使用 ActionMailer::MessageDelivery 对象上的 message 方法来访问它。
下面是上述 Rails 控制台示例中 MessageDelivery 对象的示例：

```text
irb> UserMailer.with(user: user).weekly_summary
#<ActionMailer::MailDeliveryJob:0x00007f84cb0367c0
 @_halted_callback_hook_called=nil,
 @_scheduled_at_time=nil,
 @arguments=
  ["UserMailer",
   "welcome_email",
   "deliver_now",
   {:params=>
     {:user=>
       #<User:0x00007f84c9327198
        id: 1,
        name: "Bhumi",
        email: "hi@gmail.com",
        login: "Bhumi",
        created_at: Thu, 06 Jun 2024 17:43:44.424064000 UTC +00:00,
        updated_at: Thu, 06 Jun 2024 17:43:44.424064000 UTC +00:00>},
    :args=>[]}],
 @exception_executions={},
 @executions=0,
 @job_id="07747748-59cc-4e88-812a-0d677040cd5a",
 @priority=nil,
```

## 3 多部分电子邮件和附件

multipart MIME 类型表示由多个部分组成的文档，每个部分都可能有自己独立的 MIME 类型（如 text/html 和 text/plain ）。 multipart 类型封装了在一次事务中同时发送多个文件的功能，例如在电子邮件中附加多个文件。

### 3.1 添加附件

只需将文件名和内容传递给附件方法，即可使用 Action Mailer 添加附件。Action Mailer 会`自动猜测 mime_type` ，设置 encoding 并创建附件。

```ruby
attachments["filename.jpg"] = File.read("/path/to/filename.jpg")
```
当 mail 方法被触发时，它将发送一封带有附件的多部分电子邮件，正确嵌套的顶层为 multipart/mixed ，第一部分为 multipart/alternative ，包含纯文本和 HTML 电子邮件信息。

发送附件的另一种方法是指定文件名、MIME 类型和编码头以及内容。Action Mailer 将使用传入的设置。

```ruby
encoded_content = SpecialEncode(File.read("/path/to/filename.jpg"))
attachments["filename.jpg"] = {
  mime_type: "application/gzip",
  encoding: "SpecialEncoding",
  content: encoded_content
}
```

{: .note :}
Action Mailer 会自动对附件进行 Base64 编码。如果你需要不同的设置，可以对内容进行编码，并将编码后的内容和编码以 Hash 的形式传递给 attachments 方法。如果指定了编码，Action Mailer 将不会尝试对附件进行 Base64 编码。

### 3.2 制作内联 inline 附件

有时，您可能希望以内联方式发送附件（如图片），这样它就会出现在电子邮件正文中。
为此，首先要调用 `#inline` 将附件变成内联附件：

```ruby
def welcome
  attachments.inline["image.jpg"] = File.read("/path/to/image.jpg")
end
```
然后，在视图中，可以引用 attachments 作为哈希值，并指定要内联显示的文件。
在哈希值上调用 url 并将结果传入 image_tag 方法：

```ruby
<p>Hello there, this is the image you requested:</p>

<%= image_tag attachments['image.jpg'].url %>
```
由于这是对 image_tag 的标准调用，因此也可以在附件 URL 后传递一个选项哈希值：

```ruby
<p>Hello there, this is our image</p>

<%= image_tag attachments['image.jpg'].url, alt: 'My Photo', class: 'photos' %>
```

### 3.3 多部分邮件

正如 "创建邮件视图 "中演示的那样，如果同一操作有不同的模板，Action Mailer 会自动发送多部分邮件。
例如，如果 UserMailer 中包含 welcome_email.text.erb 和 welcome_email.html.erb 中的 app/views/user_mailer ，Action Mailer 就会自动发送包含 HTML 和文本两个独立部分的多部分邮件。
[`Mail gem`](https://github.com/mikel/mail) 有辅助方法为 text/plain 和 text/html [MIME 类型](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)创建 multipart/alternate 电子邮件，也可以手动创建任何其他类型的 MIME 电子邮件。

{: .important :}
插入部件的顺序由 ActionMailer::Base.default 方法中的 :parts_order 决定。

## 4 邮件视图和布局

Action Mailer 使用视图文件来指定邮件中要发送的内容。邮件视图默认位于 app/views/name_of_mailer_class 目录中。与控制器视图类似，文件名与邮件发送器方法的名称一致。
邮件视图在布局中呈现，与控制器视图类似。邮件布局位于 app/views/layouts 目录中。默认布局为 mailer.html.erb 和 mailer.text.erb 。

### 4.1 配置自定义视图路径

可以通过多种方式更改操作的默认邮件视图。mail 方法有 template_path 和 template_name 选项：

```ruby
  def welcome_email
    @user = params[:user]
    @url  = "http://example.com/login"
    mail(to: @user.email_address,
         subject: "Welcome to My Awesome Site",
         template_path: "notifications",
         template_name: "hello")
  end
```
以上配置将 mail 方法配置为在 app/views/notifications 目录中查找名称为 hello 的模板。也可以为 template_path 指定一个`路径数组`，它们将按顺序被搜索。
如果需要更多灵活性，也可以传递一个块并呈现特定模板。不使用模板文件，直接以内联方式呈现纯文本：

```ruby
  def welcome_email
    @user = params[:user]
    @url  = "http://example.com/login"
    mail(to: @user.email_address,
         subject: "Welcome to My Awesome Site") do |format|
      format.html { render "another_template" }
      format.text { render plain: "hello" }
    end
  end
```
这将渲染 HTML 部分的模板 another_template.html.erb 和文本部分的 "hello"。呈现方法与 Action Controller 中使用的方法相同，因此可以使用所有相同的选项，如 :plain 、 :inline 等。

最后，如果需要渲染位于默认 app/views/mailer_name/ 目录之外的模板，可以像这样应用 prepend_view_path 方法：

```ruby
class UserMailer < ApplicationMailer
  prepend_view_path "custom/path/to/mailer/view"

  # This will try to load "custom/path/to/mailer/view/welcome_email" template
  def welcome_email
    # ...
  end
end
```

### 4.2 在 Action Mailer 视图中生成 URL

要在邮件程序中添加 URL，首先需要将 host 值设置为应用程序的域。这是因为与控制器不同，邮件程序实例不具备任何有关传入请求的上下文。
可以在 config/application.rb 中配置整个应用程序的默认 host 值：

```ruby
config.action_mailer.default_url_options = { host: "example.com" }
```
配置 host 后，建议电子邮件视图使用带有完整 URL 的 `*_url` ，而不是带有相对 URL 的 `*_path`助手。由于电子邮件客户端没有网络请求上下文，因此 `*_path` 助手没有基础 URL，无法形成完整的网址。

例如，不要使用：

```ruby
<%= link_to 'welcome', welcome_path %>
```
应该使用：

```ruby
<%= link_to 'welcome', welcome_url %>
```
使用完整的 URL，链接就能在电子邮件中正常工作。

默认情况下， [`url_for`](https://api.rubyonrails.org/v8.0.1/classes/ActionView/RoutingUrlFor.html#method-i-url_for) 助手会在模板中生成完整的 URL。

如果没有全局配置 :host 选项，则需要将其传递给 url_for 。

```ruby
<%= url_for(host: 'example.com',
            controller: 'welcome',
            action: 'greeting') %>
```
与其他 URL 类似，在电子邮件中也需要使用命名路由助手的 `*_url` 变体。要么全局配置 :host 选项，要么确保将其传递给 URL 助手：

```ruby
<%= user_url(@user, host: 'example.com') %>
```

### 4.3 在Action Mailer views 视图中添加图片

要在邮件中使用 image_tag 助手，需要指定 :asset_host 参数。这是因为邮件程序实例不具备任何有关传入请求的上下文。

通常 :asset_host 在整个应用程序中都是一致的，因此可以在 config/application.rb 中进行全局配置：

```ruby
config.action_mailer.asset_host = "http://example.com"
```
{: .note :}
由于无法从请求中推断协议，因此需要在 :asset_host 配置中指定 http:// 或 https:// 等协议。

现在，可以在电子邮件中显示图片了。

```ruby
<%= image_tag 'image.jpg' %>
```

### 4.4 缓存邮件视图

可以使用 cache 方法在邮件视图中执行片段缓存，这与应用程序视图类似。

```ruby
<% cache do %>
  <%= @company.name %>
<% end %>
```
要使用这一功能，您需要在应用程序的 `config/environments/*.rb` 文件中启用它：

```ruby
config.action_mailer.perform_caching = true
```
### 4.5 Action Mailer Layouts 布局

就像控制器布局一样，也可以有邮件布局。邮件布局位于 app/views/layouts 中。 以下是默认布局：

```ruby
# app/views/layouts/mailer.html.erb
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <style>
      /* Email styles need to be inline */
    </style>
  </head>

  <body>
    <%= yield %>
  </body>
</html>
```

上述布局在文件在 mailer.html.erb 中。默认布局名称在 ApplicationMailer 中指定，正如我们之前在生成邮件程序部分看到的 layout
"mailer" 行。与控制器布局类似，可以使用 yield 来渲染布局内的邮件视图。
要为指定邮件使用不同的布局，调用 [layout](https://api.rubyonrails.org/v8.0.1/classes/ActionView/Layouts/ClassMethods.html#method-i-layout) ：

```ruby
class UserMailer < ApplicationMailer
  layout "awesome" # Use awesome.(html|text).erb as the layout
end
```
要为指定邮件使用特定布局，可在格式块内的呈现调用中加入 layout: 'layout_name' 选项：

```ruby
class UserMailer < ApplicationMailer
  def welcome_email
    mail(to: params[:user].email) do |format|
      format.html { render layout: "my_layout" }
      format.text
    end
  end
end
```
上述操作将使用 my_layout.html.erb 文件渲染 HTML 部分，而使用通常的 user_mailer.text.erb 文件渲染文本部分。

## 5 发送电子邮件

### 5.1 向多个收件人发送电子邮件

通过将 :to 字段设置为电子邮件地址列表，可以向多个收件人发送电子邮件。电子邮件列表可以是一个数组，也可以是用逗号分隔的单个字符串。

```ruby
class AdminMailer < ApplicationMailer
  default to: -> { Admin.pluck(:email) },
          from: "notification@example.com"

  def new_registration(user)
    @user = user
    mail(subject: "New User Signup: #{@user.email}")
  end
end
```
通过分别设置 :cc 和 :bcc 键（与 :to 字段类似），可使用相同格式添加多个抄送 (cc) 和盲抄送 (bcc) 收件人。

### 5.2 发送带有姓名的电子邮件

除电子邮件地址外，还可以显示收件人或发件人的姓名。要在对方收到邮件时显示其姓名，可以在 to: 中使用 email_address_with_name 方法：

```ruby
def welcome_email
  @user = params[:user]
  mail(
    to: email_address_with_name(@user.email, @user.name),
    subject: "Welcome to My Awesome Site"
  )
end
```
from: 中的相同方法也能显示发件人姓名：

```ruby
class UserMailer < ApplicationMailer
  default from: email_address_with_name("notification@example.com", "Example Company Notifications")
end
```
如果名称为空（ nil 或空字符串），则返回电子邮件地址。

### 5.3 发送带翻译的电子邮件

如果没有向邮件方法传递主题，Action Mailer 会尝试在翻译中找到它。


### 5.4 不使用模板渲染发送电子邮件

在某些情况下，可能希望跳过模板渲染步骤，而是以字符串形式提供电子邮件正文。
可以使用 :body 选项来实现这一目的。请记住设置 :content_type 选项，例如下面的 text/html 。Rails 默认将 text/plain 作为内容类型。

```ruby
class UserMailer < ApplicationMailer
  def welcome_email
    mail(to: params[:user].email,
         body: params[:email_body],
         content_type: "text/html",
         subject: "Already rendered!")
  end
end
```

### 5.5 使用动态发送选项发送邮件

如果希望在发送邮件时覆盖默认的[发送配置](https://guides.rubyonrails.org/action_mailer_basics.html#action-mailer-configuration)（如 SMTP 凭据），
可以在邮件发送器动作中使用 delivery_method_options 来实现。

```ruby
class UserMailer < ApplicationMailer
  def welcome_email
    @user = params[:user]
    @url  = user_url(@user)
    delivery_options = { user_name: params[:company].smtp_user,
                         password: params[:company].smtp_password,
                         address: params[:company].smtp_host }
    mail(to: @user.email,
         subject: "Please see the Terms and Conditions attached",
         delivery_method_options: delivery_options)
  end
end
```

## 6 Action Mailer 回调方法

Action Mailer 允许指定 before_action 、 after_action 和 around_action 来配置邮件，并指定 before_deliver 、 after_deliver 和 around_deliver 来控制邮件的发送。

回调可以通过邮件发送器类中代表方法名称的块或符号来指定，这与其他回调（控制器或模型中的回调）类似。
下面是一些在邮件发送器中使用这些回调的示例。

### 6.1 before_action

可以使用 before_action 设置实例变量、使用默认值填充邮件对象或插入默认的标题和附件。

```ruby
class InvitationsMailer < ApplicationMailer
  before_action :set_inviter_and_invitee
  before_action { @account = params[:inviter].account }

  default to:       -> { @invitee.email_address },
          from:     -> { common_address(@inviter) },
          reply_to: -> { @inviter.email_address_with_name }

  def account_invitation
    mail subject: "#{@inviter.name} invited you to their Basecamp (#{@account.name})"
  end

  def project_invitation
    @project    = params[:project]
    @summarizer = ProjectInvitationSummarizer.new(@project.bucket)

    mail subject: "#{@inviter.name.familiar} added you to a project in Basecamp (#{@account.name})"
  end

  private
    def set_inviter_and_invitee
      @inviter = params[:inviter]
      @invitee = params[:invitee]
    end
end
```
### 6.2 after_action

可以使用 after_action 回调，其设置与 before_action 类似，但也可以访问在邮件发送器操作中设置的实例变量。
还可以使用 after_action 通过更新 mail.delivery_method.settings 来覆盖投递方式设置。

```ruby
class UserMailer < ApplicationMailer
  before_action { @business, @user = params[:business], params[:user] }

  after_action :set_delivery_options,
               :prevent_delivery_to_guests,
               :set_business_headers

  def feedback_message
  end

  def campaign_message
  end

  private
    def set_delivery_options
      # You have access to the mail instance,
      # @business and @user instance variables here
      if @business && @business.has_smtp_settings?
        mail.delivery_method.settings.merge!(@business.smtp_settings)
      end
    end

    def prevent_delivery_to_guests
      if @user && @user.guest?
        mail.perform_deliveries = false
      end
    end

    def set_business_headers
      if @business
        headers["X-SMTPAPI-CATEGORY"] = @business.code
      end
    end
end
```
### 6.3 after_deliver

可以使用 after_deliver 来记录邮件的发送情况。它还允许类似观察者/拦截器的行为，但可以访问完整的邮件上下文。

```ruby
class UserMailer < ApplicationMailer
  after_deliver :mark_delivered
  before_deliver :sandbox_staging
  after_deliver :observe_delivery

  def feedback_message
    @feedback = params[:feedback]
  end

  private
    def mark_delivered
      params[:feedback].touch(:delivered_at)
    end

    # An Interceptor alternative.
    def sandbox_staging
      message.to = ["sandbox@example.com"] if Rails.env.staging?
    end

    # A callback has more context than the comparable Observer example.
    def observe_delivery
      EmailDelivery.log(message, self.class, action_name, params)
    end
end
```
如果 body 被设置为非零值，邮件回调就会终止进一步处理。 before_deliver 可以通过 throw :abort 终止。

## 7 Action Mailer View Helpers

Action Mailer 视图可以使用与普通视图相同的大部分辅助方法。
在 ActionMailer::MailHelper 中还有一些 Action Mailer 特有的辅助方法。例如，这些方法允许使用 mailer 从视图访问邮件程序实例，并使用 message 访问邮件：

```ruby
<%= stylesheet_link_tag mailer.name.underscore %>
<h1><%= message.subject %></h1>
```

## 8 Action Mailer 配置

有关各种配置选项的更多详情，请参阅[《配置 Rails 应用程序》指南](https://guides.rubyonrails.org/configuring.html#configuring-action-mailer)。可以在特定环境文件（如 production.rb）中指定配置选项。

### 8.1 邮件发送器配置示例

下面是一个使用 :sendmail 发送方法的示例，添加到 config/environments/$RAILS_ENV.rb 文件中：

```ruby
config.action_mailer.delivery_method = :sendmail
# Defaults to:
# config.action_mailer.sendmail_settings = {
#   location: '/usr/sbin/sendmail',
#   arguments: %w[ -i ]
# }
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true
config.action_mailer.default_options = { from: "no-reply@example.com" }
```

### Gmail 的 Action Mailer 配置

将此添加到 config/environments/$RAILS_ENV.rb 文件，以便通过 Gmail 发送：

```ruby
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address:         "smtp.gmail.com",
  port:            587,
  domain:          "example.com",
  user_name:       Rails.application.credentials.dig(:smtp, :user_name),
  password:        Rails.application.credentials.dig(:smtp, :password),
  authentication:  "plain",
  enable_starttls: true,
  open_timeout:    5,
  read_timeout:    5 }
```

{: important :}
谷歌会阻止安全性较低的应用程序登录。可以更改 Gmail 设置，允许这种尝试。如果 Gmail 账户已启用双因素身份验证，那么需要设置一个应用程序密码，并用它代替常规密码。

## 9 预览和测试邮件

通过访问一个特殊的 Action Mailer 预览 URL，可以直观地预览已渲染的电子邮件模板。
要为 UserMailer 设置预览，在 test/mailers/previews/ 目录中创建一个名为 UserMailerPreview 的类。
要从 UserMailer 查看 welcome_email 的预览，在 UserMailerPreview 中实现一个同名的方法并调用 UserMailer.welcome_email ：

```ruby
# Preview all emails at http://localhost:3000/rails/mailers/user_mailer
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    UserMailer.with(user: User.first).welcome_email
  end
end
```

现在，预览将在 http://localhost:3000/rails/mailers/user_mailer/welcome_email 上提供。
如果在 app/views/user_mailer/welcome_email.html.erb 的邮件视图或邮件本身中更改了某些内容，预览将自动更新。预览列表也可在 http://localhost:3000/rails/mailers 中查看。


默认情况下，这些预览类位于 test/mailers/previews 中。可以使用 preview_paths 选项进行配置。例如，如果要添加 lib/mailer_previews ，可以在 config/application.rb 中配置：


```ruby
config.action_mailer.preview_paths << "#{Rails.root}/lib/mailer_previews"
```

### 9.2 错误处理

邮件发送方法中的救援块不能救援在渲染之外发生的错误。例如，后台作业中的记录反序列化错误，或来自第三方邮件发送服务的错误。
要挽救在邮件发送过程中发生的错误，使用 `rescue_from`：

```ruby
class NotifierMailer < ApplicationMailer
  rescue_from ActiveJob::DeserializationError do
    # ...
  end

  rescue_from "SomeThirdPartyService::ApiError" do
    # ...
  end

  def notify(recipient)
    mail(to: recipient, subject: "Notification")
  end
end
```

## 10 拦截和观察邮件

Action Mailer 为邮件观察器和拦截器方法提供了钩子。通过这些钩子，可以注册在每封邮件发送生命周期中被调用的类。

### 10.1 拦截邮件

通过拦截器，可以在邮件发送给`投递代理`之前对邮件进行修改。拦截器类必须实现 .delivering_email(message) 方法，该方法将在邮件发送前被调用。

拦截器需要使用 interceptors 配置选项注册。可以在初始化文件（如 config/initializers/mail_interceptors.rb ：

```ruby
Rails.application.configure do
  if Rails.env.staging?
    config.action_mailer.interceptors = %w[SandboxEmailInterceptor]
  end
end
```

{: .note :}
上面的示例使用了一个名为 "staging "的自定义环境，该环境类似于生产服务器，但用于测试目的。

### 10.2 观察邮件

邮件发送后，观察者可以访问邮件信息。观察者类必须实现 :delivered_email(message) 方法，该方法将在邮件发送后被调用。

```ruby
class EmailDeliveryObserver
  def self.delivered_email(message)
    EmailDelivery.log(message)
  end
end
```
与拦截器类似，必须使用 observers 配置选项注册观察者。在初始化文件（如 config/initializers/mail_observers.rb ：

```ruby
Rails.application.configure do
  config.action_mailer.observers = %w[EmailDeliveryObserver]
end
```
[https://github.com/memorycancel/rails-8-demo/pull/11](https://github.com/memorycancel/rails-8-demo/pull/11)
