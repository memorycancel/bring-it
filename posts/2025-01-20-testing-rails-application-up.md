---
title: 使用 Rails 8 默认测试库 minitest（上：单元测试）
layout: home
---

# 使用 Rails 8 默认测试库 minitest（上：单元测试）

2025-01-20 24:00

## 1 为什么要编写测试？

与通过浏览器或控制台进行手动测试相比，编写自动化测试能更快地确保代码按预期运行。失败的测试能迅速暴露问题，在开发过程中尽早发现并修复错误。这种做法不仅能提高代码的可靠性，还能增强对更改代码的信心。

Rails 让编写测试变得简单。

## 2 测试简介

使用 Rails，从创建新应用程序开始，测试就是开发流程的核心。

### 2.1 测试设置

一旦使用 bin/rails new application_name 创建了 Rails 项目，Rails 就会为你创建一个 test 目录。如果列出该目录的内容，你会看到

```ruby
ls -F test
application_system_test_case.rb
controllers/
helpers/
mailers/
system/
fixtures/
integration/
models/
test_helper.rb
```
### 2.2 测试目录

helpers 、 mailers 和 models 目录分别存储视图助手、邮件发送器和模型的测试。

controllers 目录用于与控制器、路由和视图相关的测试，其中将模拟 HTTP 请求并测试结果是否符合预期。

integration 目录用于包含控制器之间交互的测试。

system 测试目录存放系统测试，用于对应用程序进行全浏览器测试。系统测试可让以用户体验的方式测试应用程序，并帮助测试 JavaScript。系统测试继承自 [Capybara](https://github.com/teamcapybara/capybara)，为应用程序执行浏览器内测。

Fixures 是在测试中模拟数据的一种方法，这样就不必使用 "真实 "数据。它们存储在 fixtures 目录中。

首次生成 jobs 时，也会为作业测试创建一个 jobs 目录。test_helper.rb 文件包含测试的默认配置。

application_system_test_case.rb 表示系统测试的默认配置。

### 2.3 测试环境

默认情况下，每个 Rails 应用程序都有三个环境：开发环境 development 、测试环境 test 和生产环境 production。

每个环境的配置都可以进行类似的修改。在 test 环境下下，可以通过更改 config/environments/test.rb 中的选项来修改测试环境。

{: .note :}
运行测试 Rails 将自动切换到 RAILS_ENV=test 环境下运行。

### 2.4 编写第一个测试

bin/rails generate model 命令。在创建模型的同时，该命令还会在 test 目录中创建一个测试文件：

```ruby
bin/rails generate model post title:string body:text
...
create  app/models/post.rb
create  test/models/post_test.rb
...
```
test/models/post_test.rb 中的默认测试存根如下所示：

```ruby
require "test_helper"

class PostTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```

逐行检查该文件将有助于熟悉 Rails 测试代码和术语。
require test_helper.rb 文件时，将加载运行测试的默认配置。当包含该文件时，添加到该文件中的所有方法在测试中也都可用。

```ruby
class PostTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```
这就是所谓的测试用例，因为 PostTest 类继承自 ActiveSupport::TestCase 。 因此，它也可以使用 ActiveSupport::TestCase 中的所有方法。在本文稍后部分将看到它所提供的一些方法。

在从 Minitest::Test （即 ActiveSupport::TestCase 的超类）继承的类中定义的任何以 `test_` 开头的方法都被简单地称为测试。因此，定义为 test_password 和 test_valid_password 的方法都是测试名称，在运行测试用例时会自动运行。

Rails 还添加了一个 test 方法，它接收一个测试名称和一个代码块。它会生成一个标准的 Minitest::Unit 测试，方法名称以 `test_` 为前缀，这样你就可以专注于编写测试逻辑，而不必考虑方法的命名。例如：

```ruby
test "the truth" do
  assert true
end
```
这等价于下面这种写法：

```ruby
def test_the_truth
  assert true
end
```
虽然仍然可以使用常规的方法定义，但使用 test 宏（第一种写法）可以使测试名称更易读。
测试的这一部分称为 "断言"：

```ruby
assert true
```
断言是一行代码，用于评估对象（或表达式）的预期结果。例如，断言可以检查：

+ 这个值等于那个值吗？
+ 此对象是否为空 nil？
+ 这行代码是否抛出异常？
+ 用户密码是否大于 5 个字符？

每个测试都可以包含一个或多个断言，不限制断言的数量。只有当所有断言都成功时，测试才能通过。

要查看如何报告测试失败，可以在 post_test.rb 测试用例中添加一个失败测试。在这个示例中，断言如果不满足某些条件，文章将不会保存；因此，如果文章保存成功，测试将失败，表明测试失败。

```ruby
require "test_helper"

class PostTest < ActiveSupport::TestCase
  test "should not save post without title" do
    post = Post.new
    assert_not post.save
  end
end
```

下面是运行新添加的测试时的输出结果：

```shell
$ bin/rails test test/models/post_test.rb
Running 1 tests in a single process (parallelization threshold is 50)
Run options: --seed 36591

# Running:

F

Failure:
PostTest#test_should_not_save_post_without_title [test/models/post_test.rb:6]:
Expected true to be nil or false

bin/rails test test/models/post_test.rb:4

Finished in 0.212484s, 4.7062 runs/s, 4.7062 assertions/s.
1 runs, 1 assertions, 1 failures, 0 errors, 0 skips
```
在输出中， F 表示测试失败。 Failure 下的部分包括失败测试的名称，随后是堆栈跟踪和显示断言实际值和预期值的信息。默认的断言信息只提供了帮助识别错误的足够信息。为提高可读性，每个断言都允许使用一个可选的消息参数来定制失败消息，如下所示：

```ruby
  test "should not save post without title" do
    post = Post.new
    assert_not post.save, "Saved the post without a title"
  end
```

运行该测试后，断言信息更加友好：

```shell
Failure:
PostTest#test_should_not_save_post_without_title [test/models/post_test.rb:6]:
Saved the post without a title
```
为使该测试通过，可为 title 字段添加模型级验证。

```ruby
class Post < ApplicationRecord
  validates :title, presence: true # 添加此行进行 title 的非空验证

  has_many :comments
  has_rich_text :content
end
```

```shell
$ bin/rails test test/models/post_test.rb
Running 1 tests in a single process (parallelization threshold is 50)
Run options: --seed 36720

# Running:

.

Finished in 0.211896s, 4.7193 runs/s, 4.7193 assertions/s.
1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```
显示的小绿点表示测试已成功通过。

{: .important :}
一旦遇到任何错误或断言失败，每个测试方法的执行就会立即停止。所有测试方法都以随机顺序执行。 config.active_support.test_order 选项可用于配置测试顺序。

当测试失败时，会看到相应的回溯。默认情况下，Rails 会过滤回溯，只打印与应用程序相关的行。这样可以消除噪音，帮助您专注于代码。但是，如果您希望看到完整的回溯，请设置 -b （或 --backtrace ）参数以启用此行为：

```shell
bin/rails test -b test/models/post_test.rb
```
如果希望检查是否存在错误可以使用 assert_raises，如下所示：

```ruby
test "should report error" do
  # some_undefined_variable is not defined elsewhere in the test case
  assert_raises(NameError) do
    some_undefined_variable
  end
end
```

### 2.5 Minitest 断言

断言是测试的基石。它们实际执行检查以确保事情按计划进行。

下面是断言的摘录，你可以使用 [minitest](https://github.com/minitest/minitest) 这个 Rails 默认的测试库。 `[msg]`` 参数是一个可选的字符串信息，可以指定它来使测试失败信息更清晰。

| Assertion 断言                            | Purpose  目的                             | 
|:-----------------------------------------|:-----------------------------------------|
| assert(test, [msg])                      | 确保 test 为真。                           |
| assert_not(test, [msg])                  | 确保 test 为假。                           |
| assert_equal(expected, actual, [msg])    | 确保 expected == actual 为真。             |
| assert_not_equal(expected, actual, [msg]) | 确保 expected != actual 为真。|
| assert_same(expected, actual, [msg])  | 确保 expected.equal?(actual) 为真。|
| assert_not_same(expected, actual, [msg])| 确保 expected.equal?(actual) 为假。|
| assert_nil(obj, [msg])  | 确保 obj.nil? 为真。|
| assert_not_nil(obj, [msg])  | 确保 obj.nil? 为假。|
| assert_empty(obj, [msg])  | 确保 obj 是 empty? |
| assert_not_empty(obj, [msg])  | 确保 obj 不是 empty? 。|
| assert_match(regexp, string, [msg]) | 确保字符串与正则表达式匹配。|
| assert_no_match(regexp, string, [msg])  | 确保字符串与正则表达式不匹配。|
| assert_includes(collection, obj, [msg]) | 确保 obj 位于 collection 中。|
| assert_not_includes(collection, obj, [msg]) | 确保 obj 不在 collection 中。|
| assert_in_delta(expected, actual, [delta], [msg]) | 确保数字 expected 和 actual 相距在 delta 以内。|
| assert_not_in_delta(expected, actual, [delta], [msg]) | 确保数字 expected 和 actual 不在 delta 范围内。|
| assert_in_epsilon(expected, actual, [epsilon], [msg]) | 确保数字 expected 和 actual 的相对误差小于 epsilon 。|
| assert_not_in_epsilon(expected, actual, [epsilon], [msg]) | 确保数字 expected 和 actual 的相对误差不小于 epsilon 。|
| assert_throws(symbol, [msg]) { block }  | 确保给定代码块抛出（错误、异常）符号。|
| assert_raises(exception1, exception2, ...) { block }  | 确保给定代码块引发给定异常之一。|
| assert_instance_of(class, obj, [msg]) | 确保 obj 是 class 的实例 .|
| assert_not_instance_of(class, obj, [msg]) | 确保 obj 不是 class 的实例。|
| assert_kind_of(class, obj, [msg]) | 确保 obj 是 class 的实例或从 class 继承而来。|
| assert_not_kind_of(class, obj, [msg]) | 确保 obj 不是 class 的实例，也不是 class 的后代。|
| assert_respond_to(obj, symbol, [msg]) | 确保 obj 响应 symbol 。例如： `:200`|
| assert_not_respond_to(obj, symbol, [msg]) | 确保 obj 不响应 symbol 。|
| assert_operator(obj1, operator, [obj2], [msg])  | 确保 obj1.operator(obj2) 为真。|
| assert_not_operator(obj1, operator, [obj2], [msg])  | 确保 obj1.operator(obj2) 为假。|
| assert_predicate(obj, predicate, [msg]) | 确保 obj.predicate 为真，例如 assert_predicate str, :empty?|
| assert_not_predicate(obj, predicate, [msg]) | 确保 obj.predicate 为假，例如 assert_not_predicate str, :empty?|
| assert_error_reported(class) { block }  | 确保已报告错误类别，例如 assert_error_reported IOError { Rails.error.report(IOError.new("Oops")) }|
| assert_no_error_reported { block }  | 确保没有错误报告，例如 assert_no_error_reported { perform_service }|
| flunk([msg])  | 确保失败。这对于明确标记尚未完成的测试非常有用。|

以上是 minitest 支持的断言子集。有关详尽的最新列表，请查阅 [minitest API 文档](http://docs.seattlerb.org/minitest/Minitest)，特别是 [Minitest::Assertions](http://docs.seattlerb.org/minitest/Minitest/Assertions.html) 。

使用 minitest，可以添加自己的断言。事实上，Rails 正是这么做的。它包含一些专门的断言，让你的生活更轻松。

{: .important :}
本文不会深入讨论创建自己的断言这一主题。

### 2.7 Rails 特有的断言

`Rails 8` 为 minitest 框架添加了一些自定义断言：

| Assertion 断言                            | Purpose  目的                             | 
|:-----------------------------------------|:-----------------------------------------|
| [assert_difference(expressions, difference = 1, message = nil) {...}](https://api.rubyonrails.org/v8.0.1/classes/ActiveSupport/Testing/Assertions.html#method-i-assert_difference))| 测试表达式的返回值与 yielded 代码块中评估结果之间的数值差异。|
| [assert_no_difference(expressions, message = nil, &block)]()| 断言在调用传入代码块前后，表达式的数值运算结果没有改变。|
| [assert_changes(expressions, message = nil, from:, to:, &block)]| 测试在调用传入代码块后，表达式的求值结果是否发生变化。|
| [assert_no_changes(expressions, message = nil, &block)]| 测试在调用传入代码块后，表达式的求值结果没有改变。|
| [assert_nothing_raised { block }]| 确保给定代码块不会引发任何异常。|
| [assert_recognizes(expected_options, path, extras = {}, message = nil)]| 断言已正确处理给定路径的路由，且解析后的选项（在 expected_options hash 中给出）与路径匹配。基本上，它断言 Rails 识别了 expected_options 中给出的路径。|
| [assert_generates(expected_path, options, defaults = {}, extras = {}, message = nil)]| 断言所提供的选项可用于生成所提供的路径。这与 assert_recognizes 相反。extra 参数用于告诉请求查询字符串中其他请求参数的名称和值。message 参数允许为断言失败指定自定义错误信息。|
| [assert_routing(expected_path, options, defaults = {}, extras = {}, message = nil)]| 断言 path 和 options 双向匹配；换句话说，它验证了 path 生成 options ，然后 options 生成 path 。这实质上是将 assert_recognizes 和 assert_generates 合并为一个步骤。extras 散列允许你指定通常作为查询字符串提供给操作的选项。通过 message 参数，可以指定失败时要显示的自定义错误信息。|
| [assert_response(type, message = nil)]| 断言响应带有特定的状态代码。可以指定 :success 表示 200-299， :redirect 表示 300-399， :missing 表示 404，或 :error 以匹配 500-599 的范围。您也可以传递一个明确的状态编号或其符号等价物。更多信息，请参阅状态代码及其映射工作原理的完整列表。|
| [assert_redirected_to(options = {}, message = nil)]| 断言响应是重定向到符合给定选项的 URL。也可以传递命名路由（如 assert_redirected_to root_path ）和 Active Record 对象（如 assert_redirected_to @post ）。|
| [assert_queries_count(count = nil, include_schema: false, &block)]| 断言 &block 产生 count 次 SQL 查询。|
| [assert_no_queries(include_schema: false, &block)]| 断言 &block 不会产生任何 SQL 查询。|
| [assert_queries_match(pattern, count: nil, include_schema: false, &block)]| 断言 &block 产生的 SQL 查询与模式匹配。|
| [assert_no_queries_match(pattern, &block)]| 断言 &block 不会生成与模式匹配的 SQL 查询。|

下一章将介绍其中一些断言的用法。

### 2.7 测试用例中的断言

所有基本断言（如 Minitest::Assertions 中定义的 assert_equal ）也都可以在我们自己的测试用例中使用的类中找到。事实上，Rails 提供了以下类供你继承：

+ [ActiveSupport::TestCase](https://api.rubyonrails.org/v8.0.1/classes/ActiveSupport/TestCase.html)
+ ActionMailer::TestCase
+ ActionView::TestCase
+ ActiveJob::TestCase
+ ActionDispatch::Integration::Session
+ ActionDispatch::SystemTestCase
+ Rails::Generators::TestCase

每个类都包含 Minitest::Assertions ，允许我们在测试中使用所有基本断言。

{: .note :}
有关 minitest 的更多信息，请参阅 [minitest](http://docs.seattlerb.org/minitest) 文档。

### 2.8 Rails 测试运行器

可以使用 bin/rails test 命令一次性运行所有测试。

或者在 bin/rails test 命令中添加文件名，运行单个测试文件。这将运行单个测试文件中的所有测试方法。

还可以通过提供 -n 或 --name 标志和测试方法名称，运行测试用例中的特定测试方法。

```shell
$ bin/rails test test/models/post_test.rb -n test_the_truth
```
还可以通过提供行号，在特定行运行测试。

```shell
$ bin/rails test test/models/post_test.rb:4 # run specific test and line
```
还可以通过提供行范围来运行一系列测试。

```shell
$ bin/rails test test/models/post_test.rb:4-10 # runs tests from line 4 to 10
```

也可以通过提供目录路径来运行整个目录的测试。

```shell
bin/rails test test/models # run all tests from specific directory
```
测试运行器还提供许多其他功能，如快速失败、显示详细进度等。使用下面的命令查看测试运行器的文档：

```shell
bin/rails test -h
Usage:
  bin/rails test [PATHS...]

Run tests except system tests

Examples:
    You can run a single test by appending a line number to a filename:

        bin/rails test test/models/user_test.rb:27

    You can run multiple tests with in a line range by appending the line range to a filename:

        bin/rails test test/models/user_test.rb:10-20

    You can run multiple files and directories at the same time:

        bin/rails test test/controllers test/integration/login_test.rb

    By default test failures and errors are reported inline during a run.

minitest options:
    -h, --help                       Display this help.
        --no-plugins                 Bypass minitest plugin auto-loading (or set $MT_NO_PLUGINS).
    -s, --seed SEED                  Sets random seed. Also via env. Eg: SEED=n rake
    -v, --verbose                    Verbose. Show progress processing files.
        --show-skips                 Show skipped at the end of run.
    -n, --name PATTERN               Filter run on /regexp/ or string.
        --exclude PATTERN            Exclude /regexp/ or string from run.
    -S, --skip CODES                 Skip reporting of certain types of results (eg E).

Known extensions: rails, pride
    -w, --warnings                   Run with Ruby warnings enabled
    -e, --environment ENV            Run tests in the ENV environment
    -b, --backtrace                  Show the complete backtrace
    -d, --defer-output               Output test failures and errors after the test run
    -f, --fail-fast                  Abort test run on first failure or error
    -c, --[no-]color                 Enable color in the output
        --profile [COUNT]            Enable profiling of tests and list the slowest test cases (default: 10)
    -p, --pride                      Pride. Show your testing pride!
```

## 3 测试数据库

几乎每个 Rails 应用程序都会与数据库进行大量交互。本节将介绍如何设置测试数据库并填充示例数据。

如测试环境部分所述，每个 Rails 应用程序都有三个环境：开发环境、测试环境和生产环境。每个环境的数据库都配置在 config/database.yml 中。

专用测试数据库可单独设置测试数据并与之交互。这样，测试就可以放心地与测试数据交互，而不必担心开发或生产数据库中的数据。

## 3.1 维护测试数据库模式

为了运行测试，测试数据库需要当前的 schema。测试助手会检查测试数据库是否有任何等待迁移的数据。它会尝试将 db/schema.rb 或 db/structure.sql 加载到测试数据库中。如果迁移仍未完成，则会出现错误。这通常表明 schema 尚未完全迁移。运行迁移（使用 bin/rails db:migrate RAILS_ENV=test ）将更新 schema 。

{: .note :}
如果对现有迁移进行了修改，则需要重建测试数据库。这可以通过执行 bin/rails test:db 来完成。

## 3.2 Fixtures

要做好测试，需要考虑如何设置测试数据。在 Rails 中，可以通过定义和自定义 fixtures 来实现这一点。可以在 [Fixtures API 文档](https://api.rubyonrails.org/v8.0.1/classes/ActiveRecord/FixtureSet.html)中找到全面的文档。

Fixture 是一个花哨的词，表示一组一致的测试数据。Fixture 允许在测试运行前用预定义数据填充测试数据库。Fixture 与数据库无关，用 YAML 编写。每个 model 只有一个文件。
存储在 test/fixtures 目录中。他不是用来创建测试所需的每一个对象的，`最好只用于可应用于普通情况的默认数据`。

YAML 是一种人类可读的数据序列化语言。YAML 格式的 fixture 是描述样本数据的一种人性化方式。这类文件扩展名为 .yml（如 users.yml ）。

如果要使用关联，可以在两个不同的 fixture 之间定义一个引用节点。下面是一个 belongs_to / has_many 关联的示例：

```ruby
# test/fixtures/posts.yml
one:
  title: this is the very first post
```

```ruby
# one:
#   record: name_of_fixture (ClassOfFixture)
#   name: content
#   body: <p>In a <i>million</i> stars!</p>

first_content:
  record: first (Post)
  name: content
  body: <div>Hello, from <strong>a fixture</strong></div>
```

```ruby
# test/fixtures/comments.yml
first:
  post: first (Post)
  content: comment one

second:
  post: first (Post)
  content: comment two
```

{: .important :}
对于通过名称相互引用的关联，可以使用 fixture 名称，而不是在关联 fixture 上指定 id 属性。Rails 会自动分配一个主键，以便在运行时保持一致。有关此关联行为的更多信息，请阅读 [fixture API 文档](https://api.rubyonrails.org/v8.0.1/classes/ActiveRecord/FixtureSet.html)。

与其他 Active Record 支持的模型一样，Active Storage 附件记录继承自 ActiveRecord::Base 实例，因此可以由 fixture 填充。

考虑一个 Post 模型，它有一个作为 thumbnail 附件的关联图像，以及 fixture 数据 YAML：

```ruby
class Post < ApplicationRecord
  has_one_attached :thumbnail
end
```

```ruby
# test/fixtures/posts.yml
first:
  title: An Post
```

```ruby
# test/fixtures/active_storage/blobs.yml
first_thumbnail_blob: <%= ActiveStorage::FixtureSet.blob filename: "first.png" %>
```

```ruby
# test/fixtures/active_storage/attachments.yml
first_thumbnail_attachment:
  name: thumbnail
  record: first (Post)
  blob: first_thumbnail_blob
```

ERB 允许你在模板中嵌入 Ruby 代码。当 Rails 加载 fixtures 时，YAML 灯具格式会通过 ERB 进行预处理。这样就可以使用 Ruby 来帮助生成一些示例数据。例如，下面的代码会生成一千个用户：

```ruby
<% 1000.times do |n| %>
  user_<%= n %>:
    username: <%= "user#{n}" %>
    email: <%= "user#{n}@example.com" %>
<% end %>
```
Rails 默认从 test/fixtures 目录自动加载所有 fixtures。加载包括三个步骤:

+ 从与 fixtures 对应的表中删除任何现有数据
+ 将 fixtures 数据加载到 table 中
+ 将 fixtures 转入一个方法，以备直接访问

{: .note :}
为了从数据库中删除现有数据，Rails 会尝试禁用参照完整性触发器（如外键和检查约束）。如果在运行测试时出现权限错误，请确保数据库用户拥有在测试环境中禁用这些触发器的权限。(在 PostgreSQL 中，只有超级用户才能禁用所有触发器。请阅读 PostgreSQL 文档中有关权限的更多内容）。

fixtures 是 Active Record 的实例。如上所述，您可以直接访问该对象，因为它作为方法自动可用，其作用域仅限于测试用例。例如

```ruby
# this will return the User object for the fixture named david
users(:david)

# this will return the property for david called id
users(:david).id

# methods available to the User object can also be accessed
david = users(:david)
david.call(david.partner)
```
要一次获取多个 fixtures，可以传入一个 fixture 名称列表。例如

```ruby
# this will return an array containing the fixtures david and steve
users(:david, :steve)
```

### 3.3 Transactions  数据库事务

默认情况下，Rails 会自动将测试封装在数据库事务中，该事务一旦完成就会回滚。这使得测试彼此独立，并意味着对数据库的更改只能在单个测试中看到。
如果有多个正在写入的数据库，测试就会被封装在尽可能多的事务中，并且所有事务都会回滚。

单个测试用例可选择不使用事务回滚：

```ruby
class MyTest < ActiveSupport::TestCase
  # No implicit database transaction wraps the tests in this test case.
  self.use_transactional_tests = false
end
```

## 4 测试 models

models 测试用于测试应用程序的模型及其相关逻辑。可以使用断言和固定装置来测试这些逻辑，已经在上面的章节中进行了探讨。
Rails 模型测试存储在 test/models 目录下。Rails 提供了一个生成器来为你创建模型测试骨架。

```shell
$ bin/rails generate test_unit:model post
create  test/models/post_test.rb
```
此命令将生成以下文件：
```ruby
# post_test.rb
require "test_helper"

class PostTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```
模型测试不像 ActionMailer::TestCase 那样有自己的超类，而是继承自 ActiveSupport::TestCase 。

## 5 controllers 的功能测试

在编写功能测试时，重点是测试控制器动作如何处理请求以及预期的结果或响应。控制器功能测试有时会用于不适合系统测试的情况，如确认 API 响应。

### 5.1 功能测试的内容

+ 网络请求是否成功？
+ 用户是否被重定向到正确的页面？
+ 用户是否成功通过验证？
+ 响应中显示的信息是否正确？

```shell
$ bin/rails generate scaffold_controller post
...
create  app/controllers/posts_controller.rb
...
invoke  test_unit
create    test/controllers/posts_controller_test.rb
...
```
这将为 post 资源生成控制器代码和测试。可以查看 test/controllers 目录中的 posts_controller_test.rb 文件。

如果已经有了一个控制器，但只想为七个默认操作中的每个操作生成测试脚手架代码，可以使用以下命令：

```shell
$ bin/rails generate test_unit:scaffold post
...
invoke  test_unit
create    test/controllers/posts_controller_test.rb
...
```

{: .important :}
如果生成测试脚手架代码会看到一个 @post 值被设置并在整个测试文件中使用。这个 post 实例使用嵌套在 test/fixtures/posts.yml 文件中 :one 键内的属性。在尝试运行测试之前，请确保已在该文件中设置了键和相关值。

{: .important :}
注意：默认情况下test数据库根据 fixtures 生成假数据时，如果comment和post的key都是one，那么id会生成一样的值，这会导致外键混乱，所以应该改为post_one,comment_one.

### 5.2 用于功能测试的 HTTP 请求类型

Rails 功能测试支持 6 种请求类型：

+ get
+ post
+ patch
+ put
+ head
+ delete

所有请求类型都有可以使用的等价方法。在典型的 CRUD 应用程序中，最常使用的是 post 、 get 、 put 和 delete 。

{: .note :}
功能测试并不验证指定的请求类型是否被操作接受，而是关注结果。为测试请求类型，可使用请求测试，使测试更有目的性。


### 5.3 测试 XHR（AJAX）请求

AJAX 参考：[https://guides.rubyonrails.org/testing.html#testing-xhr-ajax-requests](https://guides.rubyonrails.org/testing.html#testing-xhr-ajax-requests)

### 5.4 测试其他请求对象

在发送请求前，将有 3 个哈希对象可供使用：

+ cookies 已设置的 cookie
+ flash   已存在于 flash 中的对象
+ session 会话变量中的对象

与普通哈希对象一样，可以通过字符串引用键来访问值。也可以通过符号名称来引用它们。例如

```ruby
flash["gordon"]               # or flash[:gordon]
session["shmession"]          # or session[:shmession]
cookies["are_good_for_u"]     # or cookies[:are_good_for_u]
```

### 5.5 实例变量

发出请求后，还可以在功能测试中访问三个实例变量：

+ @controller - 处理请求的控制器
+ @request - 请求对象
+ @response - 响应对象

```ruby
  test "should get index" do
    get posts_url
    assert_response :success

    assert_equal "index", @controller.action_name
    assert_equal "application/x-www-form-urlencoded", @request.media_type
    assert_match "Post", @response.body
  end
```

### 5.6 设置标头(Header)和 CGI 变量

HTTP 头(header)信息是与 HTTP 请求一起发送的信息，用于提供重要的元数据。CGI 变量是环境变量，用于在网络服务器和应用程序之间交换信息。HTTP 头信息和 CGI 变量可以作为头信息进行测试：

```ruby
# setting an HTTP Header
get posts_url, headers: { "Content-Type": "text/plain" } # simulate the request with custom header

# setting a CGI variable
get posts_url, headers: { "HTTP_REFERER": "http://example.com/posts" } # simulate the request with custom env variable
```

### 5.7 测试 flash 页面提示信息

从测试其他请求对象部分可以看出，在测试中可以访问的三个哈希对象之一是 flash 。
首先，应在 test_should_create_post 测试中添加一个断言：

```ruby
test "should create post" do
  assert_difference("Post.count") do
    post posts_url, params: { post: { body: @post.body, title: @post.title } }
  end

  assert_redirected_to post_url(Post.last, locale: 'zh') # 测试 locale： 默认会变成 zh
  assert_equal "Post was successfully created.", flash[:notice] # 测试flash
end
```

{: .important :}
如果使用脚手架生成器生成了控制器，那么在 create 操作中就已经实现了 Flash 消息。否则应该手动添加 flash 信息。

### 5.8 对 show 、 update 和 delete 动作的测试

```ruby
require "test_helper"

class PostsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @post = posts(:post_one)
    @user = users(:one)
    login_as @user
  end

  test "should get index" do
    # get posts_url
    get posts_url, headers: { "Content-Type": "text/plain", "HTTP_REFERER": "http://example.com/posts" }
    assert_response :success

    assert_equal "index", @controller.action_name
    # assert_equal "application/x-www-form-urlencoded", @request.media_type
    assert_match "Post", @response.body
  end

  test "should get new" do
    get new_post_url
    assert_response :success
  end

  test "should create post" do
    assert_difference("Post.count") do
      post posts_url, params: { post: { body: @post.body, title: @post.title } }
    end

    assert_redirected_to post_url(Post.last, locale: 'zh') # 测试 locale： 默认会变成 zh
    assert_equal "Post was successfully created.", flash[:notice] # 测试flash
  end

  test "should show post" do
    get post_url(@post)
    assert_response :success
  end

  test "should get edit" do
    get edit_post_url(@post)
    assert_response :success
  end

  test "should update post" do
    patch post_url(@post), params: { post: { body: @post.body, title: @post.title } }
    assert_redirected_to post_url(@post, locale: 'zh')
  end

  test "should destroy post" do
    assert_difference("Post.count", -1) do
      delete post_url(@post)
    end

    assert_redirected_to posts_url(locale: 'zh')
  end
end
```

因为设置了 auth，所以应该在 test_helper 添加登录方法：

```ruby
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

module ActiveSupport
  class TestCase
    # Run tests in parallel with specified workers
    parallelize(workers: :number_of_processors)

    # Setup all fixtures in test/fixtures/*.yml for all tests in alphabetical order.
    fixtures :all

    # Add more helper methods to be used by all tests here...
    def login_as(user)
      session = user.sessions.create!
      Current.session = session
      request = ActionDispatch::Request.new(Rails.application.env_config)
      cookies = request.cookie_jar
      cookies.signed[:session_id] = {value: session.id, httponly: true, same_site: :lax}
    end
  end
end
```

## 6 集成测试

集成测试在功能控制器测试的基础上更进一步--它们侧重于测试应用程序的多个部分如何交互，通常用于测试重要的工作流。Rails 集成测试存储在 test/integration 目录中。Rails 提供了一个生成器来创建集成测试骨架，如下所示：

```shell
$ bin/rails generate integration_test user_flows
      invoke  test_unit
      create  test/integration/user_flows_test.rb
```

```ruby
require "test_helper"

class UserFlowsTest < ActionDispatch::IntegrationTest
  # test "the truth" do
  #   assert true
  # end
end
```

这里的测试继承自 ActionDispatch::IntegrationTest 。这样，除了标准测试辅助工具外，集成测试还可以使用一些额外的辅助工具。

### 6.1 实现集成测试

{: .important :}
默认情况下，脚手架生成的测试是不包含集成（integration）测试的。

从创建一篇新博客文章的基本工作流程开始，为博客应用程序添加一个集成测试，以验证一切工作是否正常。
首先生成集成测试骨架：

```shell
$ bin/rails generate integration_test blog_flow
```
它应该已经创建了一个测试文件占位符。通过前面命令的输出，应该看到

```shell
      invoke  test_unit
      create    test/integration/blog_flow_test.rb
```
现在打开该文件并编写第一个断言：

```ruby
require "test_helper"

class BlogFlowTest < ActionDispatch::IntegrationTest
  setup do
    @post = posts(:post_one)
    @user = users(:one)
    login_as @user
  end

  test "can see the welcome page" do
    get "/"
    assert_select "h1", "文章"
  end
end
```
接着，向文章控制器的 :create 操作发出 post 请求：

```ruby
  test "can create an post" do
    get "/posts/new"
    assert_response :success

    post "/posts",
      params: { post: { title: "can create", body: "posts successfully." } }
    assert_response :redirect
    follow_redirect!
    assert_response :success
    assert_dom "p", "Post was successfully created."
  end
```

上面成功测试了访问博客和创建新文章的一个很小的工作流程。要扩展这一点，还可以针对添加评论、编辑评论或删除文章等功能添加其他测试。集成测试是试验应用程序各种用例的绝佳场所。

### 6.2 集成测试可用的辅助工具

集成测试中有许多辅助工具可供选择。其中包括

+ ActionDispatch::Integration::Runner 用于与集成测试运行器相关的助手，包括创建新会话。
+ ActionDispatch::Integration::RequestHelpers 用于执行请求。
+ ActionDispatch::TestProcess::FixtureFile 用于上传文件。
+ ActionDispatch::Integration::Session 来修改会话或集成测试的状态。


代码： [https://github.com/memorycancel/rails-8-demo/pull/9](https://github.com/memorycancel/rails-8-demo/pull/9)
