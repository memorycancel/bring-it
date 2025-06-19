---
title: 使用 Rails 8 默认测试库 minitest（下：集成测试）
layout: home
---

# 使用 Rails 8 默认测试库 minitest（下：集成测试）

2025-01-21 15:00

接上篇： [使用 Rails 8 默认测试库 minitest（上：单元测试）](2025-01-20-testing-rails-application-up)
本文继续集成测试。

## 7 系统测试

与集成测试类似，系统测试允许从用户的角度测试应用程序各组件如何协同工作。系统测试通过模拟在真实浏览器或无头浏览器（在后台运行而不打开可见窗口的浏览器）中运行测试来实现。系统测试使用 [Capybara 测试引擎](https://www.rubydoc.info/github/jnicklas/capybara)。

Rails 系统测试存储在应用程序的 test/system 目录中。要生成系统测试骨架，运行以下命令：

```shell
$ bin/rails generate system_test users
      invoke test_unit
      create test/system/users_test.rb
```
下面是刚生成的系统测试的样子：

```ruby
require "application_system_test_case"

class UsersTest < ApplicationSystemTestCase
  # test "visiting the index" do
  #   visit users_url
  #
  #   assert_selector "h1", text: "Users"
  # end
end
```

默认情况下，系统测试使用 Selenium 驱动程序、Chrome 浏览器和 1400x1400 的屏幕尺寸运行。

### 7.1 更改默认设置

Rails 让更改系统测试默认设置变得非常简单。所有设置都已抽象化，因此可以专注于编写测试。
生成新应用程序或脚手架时，会在测试目录下创建一个 application_system_test_case.rb 文件。系统测试的所有配置都应放在该文件中。
如果想更改默认设置，可以更改系统测试的 "驱动程序"。如果你想把驱动程序从 Selenium 改为 Cuprite，你可以在 Gemfile 文件中添加 [cuprite](https://github.com/rubycdp/cuprite) gem。然后在 application_system_test_case.rb 文件中执行以下操作：

```ruby
require "test_helper"
require "capybara/cuprite"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :cuprite
end
```
驱动程序名称是 driven_by 的必备参数。可传递给 driven_by 的可选参数有： :using 用于浏览器（这将只被 Selenium 使用）， :screen_size 用于更改屏幕截图的大小，以及 :options 用于设置驱动程序支持的选项。

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :firefox
end
```
如果想使用无头浏览器，可以在 :using 参数中添加 headless_chrome 或 headless_firefox 来使用无头 Chrome 浏览器或无头 Firefox 浏览器。

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome
end
```
如果要使用远程浏览器，例如 Docker 中的[无头 Chrome 浏览器](https://github.com/SeleniumHQ/docker-selenium)，则必须添加远程 url 并通过 options 将 browser 设置为远程。

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  url = ENV.fetch("SELENIUM_REMOTE_URL", nil)
  options = if url
    { browser: :remote, url: url }
  else
    { browser: :chrome }
  end
  driven_by :selenium, using: :headless_chrome, options: options
end
```
现在，应该可以连接到远程浏览器了。

```ruby
SELENIUM_REMOTE_URL=http://localhost:4444/wd/hub bin/rails test:system
```
如果你的应用程序是远程的，例如在 Docker 容器中，那么 Capybara 需要更多关于如何[调用远程服务器](https://github.com/teamcapybara/capybara#calling-remote-servers)的信息。

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  setup do
      Capybara.server_host = "0.0.0.0" # bind to all interfaces
      Capybara.app_host = "http://#{IPSocket.getaddress(Socket.gethostname)}" if ENV["SELENIUM_REMOTE_URL"].present?
    end
  # ...
end
```
现在，无论远程浏览器和服务器是在 Docker 容器还是在 CI 中运行，都能获得一个连接。
如果 Capybara 配置需要比 Rails 提供的更多设置，可以将这些附加配置添加到 application_system_test_case.rb 文件中。
有关其他设置，请参阅 [Capybara 文档](https://github.com/teamcapybara/capybara#setup)。

### 7.2 实施系统测试

如果使用了脚手架生成器，系统会自动为您创建一个系统测试骨架。如果没有使用脚手架生成器，需要从创建系统测试骨架开始。

```ruby
bin/rails generate system_test posts
```
它会创建一个测试文件占位符。通过前面命令的输出，可以看到：

```shell
      invoke  test_unit
      create    test/system/articles_test.rb
```
现在打开该文件并编写第一个断言：

```ruby
require "application_system_test_case"

class PostsTest < ApplicationSystemTestCase
  setup do
    @user = users(:one)
    login_as @user
  end

  test "viewing the index" do
    visit posts_path
    assert_selector "h1", text: "文章"
  end
end

```
测试应该能看到文章 index 页上有 h1 并通过。运行系统测试：

```shell
$ bin/rails test:system
```
{: .important :}
默认情况下，运行 bin/rails test 不会运行系统测试。确保运行 bin/rails test:system 才能真正运行它们。也可以运行 bin/rails test:all 来运行所有测试，包括系统测试。

现在可以继续测试创建新文章的流程。

```ruby
  test "should create Post" do
    visit posts_path

    click_on "New post"

    fill_in "Title", with: "Creating an Post"
    # https://stackoverflow.com/questions/10804897/how-to-fill-hidden-field-with-capybara
    # https://medium.com/eighty-twenty/testing-the-trix-editor-with-capybara-and-minitest-158f895ad15f

    # fill_in "Body", with: "Created this article successfully!" # 因为是hidden 找不到
    find(:xpath, "//*[@id='post_body']").click(with: "Created this article successfully!")

    click_on "Create Post"

    assert_text "Creating an Post"
  end
```
如果除了测试台式机外，还想测试移动设备的尺寸，可以创建另一个继承自 ActionDispatch::SystemTestCase 的类，并在测试套件中使用它。在本例中，在 /test 目录下创建了一个名为 mobile_system_test_case.rb 的文件，配置如下。

```ruby
require "test_helper"

class MobileSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :chrome, screen_size: [375, 667]
end
```
要使用此配置，在 test/system 内创建一个继承自 MobileSystemTestCase 的测试。现在可以使用多种不同的配置来测试应用程序了。

在 system 文件夹创建 mobile_posts_test.rb :

```ruby
require "mobile_system_test_case"

class MobilePostsTest < MobileSystemTestCase
  setup do
    @user = users(:one)
    login_as @user
  end

  test "visiting the index" do
    visit posts_url
    assert_selector "h1", text: "文章"
  end
end
```
执行步骤一样。
以下是 Capybara 提供的断言摘要，可用于系统测试。

| Assertion 断言                            | Purpose  目的                             | 
|:-----------------------------------------|:-----------------------------------------|
| assert_button(locator = nil, **options, &optional_filter_block) | 检查页面是否有一个带有给定文本、值或 ID 的按钮。 |
| assert_current_path(string, **options) | 断言页面具有给定的路径。|
| assert_field(locator = nil, **options, &optional_filter_block)| 检查页面是否有带有给定标签、名称或 id 的表单字段。|
| assert_link(locator = nil, **options, &optional_filter_block)| 检查页面中是否有带有给定文本或 id 的链接|
| assert_selector(*args, &optional_filter_block)| 断言页面上有给定的选择器。|
| assert_table(locator = nil, **options, &optional_filter_block| 检查页面中是否有带有给定 id 或标题的表格。|
| assert_text(type, text, **options)| 断言页面具有给定的文本内容。|

ScreenshotHelper 是一个辅助工具，用于截取测试截图。这有助于在测试失败时查看浏览器，或稍后查看截图进行调试。
提供了两种方法： take_screenshot 和 take_failed_screenshot 。 take_failed_screenshot 会自动包含在 Rails 内部的 before_teardown 中。
take_screenshot 辅助方法可以包含在测试的任何地方，以截取浏览器的屏幕截图。
系统测试与集成测试类似，都是测试用户与控制器、模型和视图之间的交互，但系统测试是以真实用户使用应用程序的方式进行测试。通过系统测试，您可以测试用户在应用程序中进行的任何操作，如评论、删除文章、发布文章草稿等。

## 8 测试助手

为避免代码重复，您可以添加自己的测试助手。
如果发现帮助程序占用了 test_helper.rb 的空间，可以将它们提取到单独的文件中。 test/lib 或 test/test_helpers 是存储它们的好地方。

可能会发现，在 test_helper.rb 中急切地require helper，这样测试文件就可以隐式地访问它们。这可以通过使用 globbing 来实现，如下所示

```ruby
# test/test_helper.rb
Dir[Rails.root.join("test", "test_helpers", "**", "*.rb")].each { |file| require file }
```

## 9 测试路由

与 Rails 应用程序中的其他功能一样，也可以对路由进行测试。路由测试存储在 test/controllers/ 中，或者是控制器测试的一部分。如果你的应用程序有复杂的路由，Rails 提供了许多有用的助手来测试它们。
有关 Rails 中可用路由断言的更多信息，请参阅 [ActionDispatch::Assertions::RoutingAssertions](https://api.rubyonrails.org/v8.0.1/classes/ActionDispatch/Assertions/RoutingAssertions.html) 的 API 文档。

## 10 测试视图

通过断言关键 HTML 元素的存在及其内容来测试对请求的响应，是测试应用程序视图的一种方法。与路由测试一样，视图测试也存储在 test/controllers/ 中，或者是控制器测试的一部分。

通过 assert_dom 和 assert_dom_equal 等方法，可以使用简单但功能强大的语法来查询响应中的 HTML 元素。

关于更多试图测试参考[https://guides.rubyonrails.org/testing.html#testing-views](https://guides.rubyonrails.org/testing.html#testing-views)

其中可以通过 ` Nokogiri` 对dom进行各种选中操作。

## 11 测试邮件发送器

与 Rails 应用程序的其他部分一样，邮件发送器类也应进行测试，以确保它们按预期运行。测试邮件发送器类的目的是确保：
+ 邮件正在处理（创建和发送）
+ 电子邮件内容正确无误（主题、发件人、正文等）
+ 在正确的时间发送正确的邮件

测试邮件程序有两个方面，即单元测试和功能测试。在单元测试中，需要使用严格控制的输入单独运行邮件程序，
并将输出与已知值（固定值）进行比较。在功能测试中，不需要测试邮件发送器产生的细节；相反，
需要测试控制器和模型是否以正确的方式使用邮件发送器。测试的目的是证明在正确的时间发送了正确的邮件。

### 11.1 单元测试

为了对邮件程序进行单元测试，fixtures 用于提供输出结果的示例。由于这些是示例邮件，而不是像其他fixtures 那样的 Active Record 
数据，因此它们被保存在自己的子目录中，与其他 fixtures 分开。 test/fixtures 中的目录名称与邮件程序的名称直接对应。
因此，对于名为 UserMailer 的邮件程序，fixtures 应位于 test/fixtures/user_mailer 目录中。
如果生成了邮件程序，生成器不会为邮件程序的操作创建存根 fixtures。必须自己创建这些文件。

下面是一个单元测试，用于测试名为 UserMailer 的邮件发送器，其操作 invite 用于向朋友发送邀请：

```ruby
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "invite" do
    # Create the email and store it for further assertions
    email = UserMailer.create_invite("me@example.com",
                                     "friend@example.com", Time.now)

    # Send the email, then test that it got queued
    assert_emails 1 do
      email.deliver_now
    end

    # Test the body of the sent email contains what we expect it to
    assert_equal ["me@example.com"], email.from
    assert_equal ["friend@example.com"], email.to
    assert_equal "You have been invited by me@example.com", email.subject
    assert_equal read_fixture("invite").join, email.body.to_s
  end
end
```
测试中创建了电子邮件，返回的对象存储在 email 变量中。第一个断言检查邮件是否已发送，然后在第二批断言中检查邮件内容。助手 read_fixture 用于从文件中读取内容。

{: .note :}
当只有一个（HTML 或文本）部分时，就会出现 email.body.to_s 。如果邮件程序同时提供这两个部分，可以使用 email.text_part.body.to_s 或 email.html_part.body.to_s 针对特定部分测试 email patials。

下面是 invite fixtures 的内容：

```text
Hi friend@example.com,

You have been invited.

Cheers!
```
config/environments/test.rb 中的 ActionMailer::Base.delivery_method = :test 行将发送方式设置为测试模式，这样电子邮件就不会被实际发送（测试时可避免向用户发送垃圾邮件）。而是将实际需要发送的列表附加到一个数组（ ActionMailer::Base.deliveries ）中。

{: .important :}
只有在 ActionMailer::TestCase 和 ActionDispatch::IntegrationTest 测试中，才会自动重置 ActionMailer::Base.deliveries 数组。如果你想在这些测试用例之外有一个干净的界面，可以用以下方法手动重置： ActionMailer::Base.deliveries.clear

可以使用 assert_enqueued_email_with 断言来确认电子邮件是否已使用所有预期的邮件发送器方法参数和/或参数化邮件发送器参数进行了排队。这样可以测试任何使用 deliver_later 方法启动的邮件。
与基本测试用例一样，创建电子邮件，并将返回的对象存储在 email 变量中。下面的示例包括传递参数和/或参数的各种变化。
此示例将断言电子邮件已使用正确参数排队：

```ruby
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "invite" do
    # Create the email and store it for further assertions
    email = UserMailer.create_invite("me@example.com", "friend@example.com")

    # Test that the email got enqueued with the correct arguments
    assert_enqueued_email_with UserMailer, :create_invite, args: ["me@example.com", "friend@example.com"] do
      email.deliver_later
    end
  end
end
```
此示例将通过传入一个名为 args 的参数哈希值，断言邮件发送器已使用名为 arguments 的正确邮件发送器方法启动：

```ruby
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "invite" do
    # Create the email and store it for further assertions
    email = UserMailer.create_invite(from: "me@example.com", to: "friend@example.com")

    # Test that the email got enqueued with the correct named arguments
    assert_enqueued_email_with UserMailer, :create_invite,
    args: [{ from: "me@example.com", to: "friend@example.com" }] do
      email.deliver_later
    end
  end
end
```

此示例将断言一个参数化的邮件发送器已使用正确的参数和参数被挂起。邮件参数以 params 的形式传递，邮件方法参数以 args 的形式传递：

```ruby
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "invite" do
    # Create the email and store it for further assertions
    email = UserMailer.with(all: "good").create_invite("me@example.com", "friend@example.com")

    # Test that the email got enqueued with the correct mailer parameters and arguments
    assert_enqueued_email_with UserMailer, :create_invite,
    params: { all: "good" }, args: ["me@example.com", "friend@example.com"] do
      email.deliver_later
    end
  end
end
```
该示例展示了另一种测试方法，即使用正确的参数对参数化的邮件程序进行查询：

```ruby
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "invite" do
    # Create the email and store it for further assertions
    email = UserMailer.with(to: "friend@example.com").create_invite

    # Test that the email got enqueued with the correct mailer parameters
    assert_enqueued_email_with UserMailer.with(to: "friend@example.com"), :create_invite do
      email.deliver_later
    end
  end
end
```
### 11.2 功能和系统测试

单元测试测试电子邮件的属性，而功能和系统测试则测试用户交互是否恰当地触发了电子邮件的发送。例如，可以检查邀请好友操作是否恰当地发送了电子邮件：

```ruby
# Integration Test
require "test_helper"

class UsersControllerTest < ActionDispatch::IntegrationTest
  test "invite friend" do
    # Asserts the difference in the ActionMailer::Base.deliveries
    assert_emails 1 do
      post invite_friend_url, params: { email: "friend@example.com" }
    end
  end
end
```

```ruby
# System Test
require "test_helper"

class UsersTest < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome

  test "inviting a friend" do
    visit invite_users_url
    fill_in "Email", with: "friend@example.com"
    assert_emails 1 do
      click_on "Invite"
    end
  end
end
```
{: .note :}
assert_emails 方法与特定的发送方法并不绑定，可以与使用 deliver_now 或 deliver_later 方法发送的电子邮件一起使用。如果想明确断言电子邮件已被挂起，可以使用 assert_enqueued_email_with 方法（上述示例）或 assert_enqueued_emails 方法。更多信息请[参阅文档](https://api.rubyonrails.org/v8.0.1/classes/ActionMailer/TestHelper.html)。

## 12 测试 jobs

可对作业进行隔离测试（侧重于作业的行为）和上下文测试（侧重于调用代码的行为）。


### 12.1 独立测试作业

生成作业时，也会在 test/jobs 目录中生成相关的测试文件。例如：
```ruby
require "test_helper"

class BillingJobTest < ActiveJob::TestCase
  test "account is charged" do
    perform_enqueued_jobs do
      BillingJob.perform_later(account, product)
    end
    assert account.reload.charged_for?(product)
  end
end
```
在调用 perform_enqueued_jobs 之前，测试的默认队列适配器不会执行作业。此外，它还会在每个测试运行前清除所有作业，这样测试就不会相互干扰。
测试使用 perform_enqueued_jobs 和 perform_later 而不是 perform_now ，这样如果配置了重试，重试失败会被测试捕获，而不是重新排队和忽略。


### 12.2 在上下文中测试作业

测试作业是否被正确挂起（例如，通过控制器操作）是一种很好的做法。 ActiveJob::TestHelper 模块提供了几种方法可以帮助实现这一点，例如 assert_enqueued_with 。下面是一个测试账户模型方法的示例：

```ruby
require "test_helper"

class AccountTest < ActiveSupport::TestCase
  include ActiveJob::TestHelper

  test "#charge_for enqueues billing job" do
    assert_enqueued_with(job: BillingJob) do
      account.charge_for(product)
    end

    assert_not account.reload.charged_for?(product)

    perform_enqueued_jobs

    assert account.reload.charged_for?(product)
  end
end
```
### 12.3 测试是否会引发异常

测试作业是否在某些情况下引发异常可能比较棘手，尤其是在配置了重试的情况下。当作业引发异常时， perform_enqueued_jobs 辅助函数会导致测试失败，因此要想在异常引发时测试成功，就必须直接调用作业的 perform 方法。

```ruby
require "test_helper"

class BillingJobTest < ActiveJob::TestCase
  test "does not charge accounts with insufficient funds" do
    assert_raises(InsufficientFundsError) do
      BillingJob.new(empty_account, product).perform
    end
    assert_not account.reload.charged_for?(product)
  end
end
```
一般情况下不推荐使用这种方法，因为它规避了框架的某些部分，例如参数序列化。

## 13 测试 Action Cable

由于 Action Cable 会在应用程序的不同级别使用，因此需要测试channels, connection以及其他实体broadcast的消息是否正确。

### 13.1 连接测试用例

默认情况下，使用 Action Cable 生成新的 Rails 应用程序时，会在 test/channels/application_cable 目录下生成基础连接类（ ApplicationCable::Connection ）的测试。

连接测试的目的是检查连接的标识符是否分配正确，或拒绝任何不正确的连接请求。下面是一个示例：

```ruby
class ApplicationCable::ConnectionTest < ActionCable::Connection::TestCase
  test "connects with params" do
    # Simulate a connection opening by calling the `connect` method
    connect params: { user_id: 42 }

    # You can access the Connection object via `connection` in tests
    assert_equal connection.user_id, "42"
  end

  test "rejects connection without params" do
    # Use `assert_reject_connection` matcher to verify that
    # connection is rejected
    assert_reject_connection { connect }
  end
end
```
也可以像在集成测试中那样指定请求 cookie：

```ruby
test "connects with cookies" do
  cookies.signed[:user_id] = "42"

  connect

  assert_equal connection.user_id, "42"
end
```
更多信息，请参阅 [ActionCable::Connection::TestCase](https://api.rubyonrails.org/v8.0.1/classes/ActionCable/Connection/TestCase.html) 的 API 文档。

### 13.2 Channel Test Case  13.2 通道测试用例

默认情况下，生成频道时，也会在 test/channels 目录下生成相关测试。下面是一个带有聊天频道的测试示例：

```ruby
require "test_helper"

class ChatChannelTest < ActionCable::Channel::TestCase
  test "subscribes and stream for room" do
    # Simulate a subscription creation by calling `subscribe`
    subscribe room: "15"

    # You can access the Channel object via `subscription` in tests
    assert subscription.confirmed?
    assert_has_stream "chat_15"
  end
end
```
该测试非常简单，仅断言通道将连接订阅到了特定的流。

```ruby
require "test_helper"

class WebNotificationsChannelTest < ActionCable::Channel::TestCase
  test "subscribes and stream for user" do
    stub_connection current_user: users(:john)

    subscribe

    assert_has_stream_for users(:john)
  end
end
```
更多信息，请参阅 [ActionCable::Channel::TestCase](https://api.rubyonrails.org/v8.0.1/classes/ActionCable/Channel/TestCase.html) 的 API 文档。

### 13.3 自定义断言和测试其他组件内的广播

Action Cable 随附了大量自定义断言，可用于减少测试的冗长度。有关可用断言的完整列表，请参阅 [ActionCable::TestHelper](https://api.rubyonrails.org/v8.0.1/classes/ActionCable/TestHelper.html) 的 API 文档。

确保在其他组件（如控制器）中广播了正确的消息是一种很好的做法。这正是 Action Cable 提供的自定义断言非常有用的地方。例如，在`model`内部：

```ruby
require "test_helper"

class ProductTest < ActionCable::TestCase
  test "broadcast status after charge" do
    assert_broadcast_on("products:#{product.id}", type: "charged") do
      product.charge(account)
    end
  end
end
```
如果要测试使用 Channel.broadcast_to 进行的广播，则应使用 Channel.broadcasting_for 生成底层流名称：

```ruby
# app/jobs/chat_relay_job.rb
class ChatRelayJob < ApplicationJob
  def perform(room, message)
    ChatChannel.broadcast_to room, text: message
  end
end
```

```ruby
# test/jobs/chat_relay_job_test.rb
require "test_helper"

class ChatRelayJobTest < ActiveJob::TestCase
  include ActionCable::TestHelper

  test "broadcast message to room" do
    room = rooms(:all)

    assert_broadcast_on(ChatChannel.broadcasting_for(room), text: "Hi!") do
      ChatRelayJob.perform_now(room, "Hi!")
    end
  end
end
```

## 14 在持续集成（CI）中运行测试

持续集成（CI）是一种开发实践，在这种实践中，变更会经常集成到主代码库中，因此在合并前会自动进行测试。

要在 CI 环境中运行所有测试，只需执行一条命令即可：


```shell
$ bin/rails test
```
如果使用的是系统测试， bin/rails test 将不会运行它们，因为它们可能会很慢。要运行它们，可添加另一个运行 bin/rails test:system 的 CI 步骤，或将第一个步骤改为 bin/rails test:all ，它将运行包括系统测试在内的所有测试。

## 15 并行测试

并行运行测试可缩短整个测试套件的运行时间。虽然分叉进程是默认方法，但也支持线程。

### 15.1 使用进程进行并行测试

默认的并行化方法是使用 Ruby 的 DRb 系统分叉进程。进程根据提供的 Worker 数量分叉。默认数量是机器上的实际核心数，但可以通过向 parallelize 方法传递的数量进行更改。

要启用并行化，请在 test_helper.rb 方法中添加以下内容：

```ruby
class ActiveSupport::TestCase
  parallelize(workers: 2)
end
```
传递的 Worker 数量是进程将被分叉的次数。如果需要本地测试套件的并行化方式与 CI 不同，因此提供了一个环境变量，以便轻松更改测试运行应使用的 Worker 数量：

```shell
PARALLEL_WORKERS=15 bin/rails test
```
在并行测试时，Active Record 会自动为每个进程创建数据库并将模式加载到数据库中。数据库将以与工作者相对应的编号作为后缀。例如，如果有 2 个工作者，测试将分别创建 test-database-0 和 test-database-1 。

如果通过的 Worker 数量为 1 或更少，进程将不会被分叉，测试也不会被并行化，而是使用原始的 test-database 数据库。

提供了两个钩子，一个在进程被分叉时运行，另一个在分叉进程关闭前运行。如果应用程序使用多个数据库或执行其他依赖于工作者数量的任务，这两个钩子可能会很有用。

parallelize_setup 方法在进程分叉后立即调用。 parallelize_teardown 方法在进程关闭前调用。

```ruby
class ActiveSupport::TestCase
  parallelize_setup do |worker|
    # setup databases
  end

  parallelize_teardown do |worker|
    # cleanup databases
  end

  parallelize(workers: :number_of_processors)
end
```

在使用线程并行测试时，不需要或不能使用这些方法。

### 15.2 使用线程进行并行测试

如果喜欢使用线程或正在使用 JRuby，提供了线程并行化选项。线程并行器由 minitest 的 Parallel::Executor 支持。
要将并行化方法改为使用线程而非叉，在 test_helper.rb ：

```ruby
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors, with: :threads)
end
```
由 JRuby 或 TruffleRuby 生成的 Rails 应用程序会自动包含 with: :threads 选项。

### 15.3 测试并行事务

当要测试在线程中运行并行数据库事务的代码时，这些线程可能会相互阻塞，因为它们已经嵌套在隐式测试事务之下。
为了解决这个问题，可以通过设置 self.use_transactional_tests = false 来禁用测试用例类中的事务：

```ruby
class WorkerTest < ActiveSupport::TestCase
  self.use_transactional_tests = false

  test "parallel transactions" do
    # start some threads that create transactions
  end
end
```
{: .important :}
禁用事务测试后，必须清理测试创建的任何数据，因为测试完成后更改不会自动回滚。

### 15.4 并行测试的阈值

并行运行测试会增加数据库 setup 和 fixtures 加载的开销。因此，Rails 不会并行执行少于 50 个测试。
可以在 test.rb  中配置这个阈值：
```ruby
config.active_support.test_parallelization_threshold = 100
```

在测试用例级别设置并行化时也是如此：

```ruby
class ActiveSupport::TestCase
  parallelize threshold: 100
end
```

## 16 测试急迫加载 Eager Loading

通常情况下，应用程序不会在 development 或 test 环境中急于加载以加快速度。但在 production 环境中会这样做。
如果项目中的某些文件由于某种原因无法加载，则必须在部署到生产环境之前进行检测。

### 16.1 持续集成

如果项目已实施 CI，那么在 CI 中进行急迫加载是确保应用程序急迫加载的一种简便方法。
CI 通常会设置一个环境变量来指示测试套件正在那里运行。例如：

```ruby
# config/environments/test.rb
config.eager_load = ENV["CI"].present?
```
从 Rails 7 开始，新生成的应用程序默认是这样配置的。如果项目没有持续集成，仍然可以通过调用 Rails.application.eager_load! ：

```ruby
require "test_helper"

class ZeitwerkComplianceTest < ActiveSupport::TestCase
  test "eager loads all files without errors" do
    assert_nothing_raised { Rails.application.eager_load! }
  end
end
```
## 17 其他测试资源

### 17.1 错误

在系统测试、集成测试和功能控制器测试中，Rails 会尝试从错误中 rescue 出来，并默认使用 HTML 错误页面进行响应。这一行为可由 [config.action_dispatch.show_exceptions](https://guides.rubyonrails.org/configuring.html#config-action-dispatch-show-exceptions) 配置控制。

### 17.2 测试依赖时间的代码

Rails 提供了内置的辅助方法，使能够断言时间敏感的代码按预期运行。下面的示例使用了 [travel_to](https://api.rubyonrails.org/v8.0.1/classes/ActiveSupport/Testing/TimeHelpers.html#method-i-travel_to) 辅助方法：

```ruby
# Given a user is eligible for gifting a month after they register.
user = User.create(name: "Gaurish", activation_date: Date.new(2004, 10, 24))
assert_not user.applicable_for_gifting?

travel_to Date.new(2004, 11, 24) do
  # Inside the `travel_to` block `Date.current` is stubbed
  assert_equal Date.new(2004, 10, 24), user.activation_date
  assert user.applicable_for_gifting?
end

# The change was visible only inside the `travel_to` block.
assert_equal Date.new(2004, 10, 24), user.activation_date
```
