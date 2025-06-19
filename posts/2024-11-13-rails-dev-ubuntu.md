---
title: 使用 Omakese 在 Ubuntu 上搭建Ruby on Rails开发环境
layout: home
---

# 使用 Omakese 在 Ubuntu 上搭建Ruby on Rails开发环境

2024-11-13 16:00

`Omakase(おまかせ)是日语词汇,在美食界通常指“厨师发办”,即客人无需点菜,完全由主厨根据当季食材、个人喜好和餐厅特色来安排菜品。`

Omakase是DHH最新花活，见 https://omakub.org/  ，旨在Ubuntu24.04系统上，搭建统一的Rails开发环境。
刚好家里有一台Dell工作站，果断装之，甚好。以下是安装时大致步骤记录和分享之。
Omakase 遵循“默认大于配置”原则，甚至有推荐的笔记本 https://frame.work/ ， 我用 Dell工作站高替。在这里只选取了大部分喜欢的“菜”浅尝，并非全部。

## 安装ubuntu

找一台可以装windows的电脑，下载UltraISO软件和 [Ubuntu 24.04 Desktop ISO镜像文件](https://ubuntu.com/download/desktop#system-requirements),准备一个U盘，将文件考入U盘，
重启电脑，F2进入Boot修改启动项目，按照自动向导安装Ubuntu。

另外准备好科学上网，rust（cargo）具体步骤略。

## 安装Omakase

到这里相当于已经走进餐厅了，遂叫“厨师”https://omakub.org/#installing-omakub，上菜。也就是执行
```bash
wget -qO- https://omakub.org/install | bash
```
命令，到这里其实并没有按照预想上完所有菜，因为“食材”（电脑、网络环境）都不一样。真是“巧妇难为无米之炊”啊～ 好在DHH主厨为我们准备了“菜谱”，我们只需要替换相应的食材，然后按照菜谱制作就好了。

## 安装基础设施

于是我挑选了几道我爱的精品菜。烹之。先准备“以下食材”

+ `nerdfonts`：书呆子Nerd自体 https://www.nerdfonts.com/ 有很多变体版本自选，装上后终端会养眼很多。
+ `mise`：高替 rvm https://mise.jdx.dev/ ，rust重写的，可以管理ruby node等等版本，包会比rvm新，还会自动检测项目里面的.ruby_version切换版本
+ `alacritty`：高替terminal https://alacritty.org/ 
+ `zellij`：高替 tmux  https://zellij.dev/
+ `neovim`：高替 vi/vim https://neovim.io/

安装自行参考官方链接，或者使用 rust（cargo）进行安装。

## 替换系统默认项

mise,neovim和字体安装好后就自动替换默认了，最后将 alacritty 设置为默认的 terminal

```
> GNOME Keyboard Settings
        > "Customize Shortcuts"
            > "Custom Shortcuts"
                > "Add shortcuts"
                    - `alacritty -e zellij`
                    - Set keybinding to terminal launch (i.e. `Ctrl+Alt+T`)    

```
将 alacritty 设置为默认的 terminal

```bash
$ cargo install zellij alacritty
$ sudo update-alternatives --install \
      /usr/bin/x-terminal-emulator x-terminal-emulator \
$(which alacritty) 50
$ sudo update-alternatives --config x-terminal-emulator
```
编辑 ` ~/.config/alacritty/alacritty.yml`
```
    shell:
        program: /usr/bin/bash
        args:
        - -l
        - -c
        - zellij attach --index 0 || zellij
```

至此，酒足饭饱，后面通过`Ctrl+Alt+T`就可以打开alacritty，通过vi打开文件夹进行coding了（具体食用方法参考https://zellij.dev/tutorials/basic-functionality/）

当然，对于`字体，包管理器，编辑器，终端`都是众口难调、老生长谈的问题，面对新的工具，我们还是要发扬“拿来主义”精神，好则留，坏则废。
这也遵循Rubyist的slogan，进步总是比稳定要好～
