---
title: 使用 mise 管理 ruby 版本的一点细节
layout: home
---

# 使用 mise 管理 ruby 版本的一点细节

2024-11-20 15:00

`mise`是新一代用rust编写的用于管理编程语言版本环境的工具。
[https://github.com/jdx/mise](https://github.com/jdx/mise)
最令人惊艳的地方是他涵盖了很多语言，有一种`less is more`的美感。

例如，node的需要`nvm`。ruby需要`rvm`。python需要`pyenv`，
而现在只需要一个`mise`就可以了。是不是很神奇？

具体使用方法这里不做赘述了，参考官网。经常使用的CLI是：
```
mise use node@20
mise use ruby@3.3.5
mise use -g ruby@3.3.6
```

这时候会有这样一个场景，在本地我有两个项目A和B，A项目是基于 ruby 的 3.3.5 版本
而B项目是基于 ruby 的 3.3.6 版本。
那么在日常开发时，在开发项目A时需要先执行
```
mise use ruby@3.3.5
```
A项目的gem包也都是在 `ruby@3.3.5` 版本下。
而切换到项目B时，需要执行：

```
mise use ruby@3.3.6
```
因为B项目包都是基于`ruby@3.3.6`的。这样一来操作十分繁琐，切换成本高，
更重要的时候还容易出错，假如忘记切换，就会将两个项目的基于不同ruby版本的包混淆。

还好，mise自有妙计。只需要在项目中添加一个 `.ruby-version`文件：

```
3.3.5
```
这样一来，通过CLI 进入到该项目时，`mise`会自动切换到相应的版本，省去了手动切换的成本，
并且规避掉忘记切换的风险。
