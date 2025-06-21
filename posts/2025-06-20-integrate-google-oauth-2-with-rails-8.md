---
title: Rails 8 集成 google oauth 2.0
layout: home
---

# Rails 8 集成 google oauth 2.0

## 1 注册 google 开发者

+ https://developers.google.com/identity/sign-in/web/sign-in?hl=zh-cn

获取 GOOGLE_CLIENT_ID 和 GOOGLE_CLIENT_SECRET

## 2 授权流程

此场景有3方角色：

+ 用户方：user
+ 网站方：rails
+ 第三方授权：google

大致流程：

1. rails 在 google 注册 oauth 应用，获取 GOOGLE_CLIENT_ID 和 GOOGLE_CLIENT_SECRET，并设置在服务端。
2. user 访问 rails 资源，发现需要注册登录，点击 google 的按钮，跳转至 google 帐号授权页面。
3. user 在google 帐号授权页面，登录 google 帐号并完成授权操作，google 跳转回 rails，触发回调接口。
4. rails 通过回调接口获得访问 google 帐号资源的权限 token，通过 token 获取用户的 email 等信息。
5. 为了与 rails 已有auth流程结合，利用 user 的 google 帐号的 email 新建帐号，并设置初始密码并登录。


## 3 集成 google oauth 2.0 相关 gem

+ [https://github.com/memorycancel/rails-8-demo/pull/16](https://github.com/memorycancel/rails-8-demo/pull/16)

期间当 rails 8 需要跳转至外链（google auth 链接）时，turbo 有一个bug，解决办法如下：

+ [hotwired turbo-rails: Redirecting to an external URL from a turbo POST request fails with "blocked by CORS policy" #483](https://github.com/hotwired/turbo-rails/issues/483)

调试过程中，加上配置参数：

```shell
GOOGLE_CLIENT_ID=xxx GOOGLE_CLIENT_SECRET=xxx rails s
```
