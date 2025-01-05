---
title: 使用 Rails 8 提供的默认缓存 Solid Cache
layout: home
---

# 使用 Rails 8 提供的默认缓存 Solid Cache

2025-01-05 18:00

缓存是提高应用程序性能的最有效方法之一。它能让运行在适度基础设施上的网站（单个服务器和单个数据库）支持成千上万的并发用户。

Rails 提供了一套开箱即用的缓存功能，不仅可以缓存数据，还可以解决缓存过期、缓存依赖和缓存失效等难题。

默认情况下，Action Controller 缓存仅在生产环境中启用。接上篇[使用 Rails 8 提供的默认任务队列 Solid Queue](2025-01-03-using-solid-queue-of-rails-8)
我们还是在开发环境“手动设置为生产环境”。

```shell
export RAILS_ENV=production
```
