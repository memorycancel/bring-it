---
title: Vagrant cheatsheet
layout: home
---

# Vagrant cheatsheet

2024-11-21 14:00

最近新出的 `rails 8.0.0` 默认部署环境使用 `kamal`，但是由于国内的运维基础设施差异性
并不兼容。于是想用vagrant + virtualbox 实现搭建一致服务器环境。好在这俩货
都是开源协议。

免费代表进步，收费就要挨打。

环境：`Ubuntu 24.04 Pro`

与之兼容的是 virtualbox 7.1： `virtualbox-7.1_7.1.4-165100~Ubuntu~noble_amd64.deb`

安装最新的vagrant：
[https://developer.hashicorp.com/vagrant/install](https://developer.hashicorp.com/vagrant/install)
报错很诡异，原来是vagrant最多兼容到virtualbox`7.0`版本...
好在报错只是对版本号进行了检查，并不是真正的API不兼容。
参考解决办法：
[https://github.com/hashicorp/vagrant/issues/13501](https://github.com/hashicorp/vagrant/issues/13501)

下面就是使用了。vagrant的最重要且常用的几条CLI：

```bash
# 获得Ubuntu 18.04 LTS 64的Vagrantfile
vagrant init hashicorp/bionic64
# 开机
vagrant up
# 进入
vagrant ssh
# 关机
vagrant halt
# 销毁
vagrant destroy
```
执行`vagrant init hashicorp/bionic64` 会生成一个 `Vagrantfile`文件：
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # 创建虚拟机的初始镜像，例如此处是用户hashicorp创建的bionic64镜像
  # 在 vagrant中这个镜像成为 box（盒子）
  # 更多盒子可以在 https://vagrantcloud.com/search 搜索，也可以自己创建
  # 这里的 bionic64镜像实际上是 Ubuntu 18.04 LTS 64
  config.vm.box = "hashicorp/bionic64"

  # 关闭镜像自动更新，使用命令`vagrant box outdated` 手动更新
  # 不推荐！
  # config.vm.box_check_update = false

  # 端口映射 host是主机，guest是主机创建的虚拟机。
  # 如果访问 127.0.0.1:8080 将会转发到虚拟机的 VBIP:80
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # 通过设置 127.0.0.1 只允许主机访问端口转发，而禁止公网访问
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # 创建一个私有网络，只允许本机通过指定 IP 访问虚拟机网络
  # config.vm.network "private_network", ip: "192.168.33.10"

  # 创建公网，通常与桥接网络相匹配。
  # 桥接网络使虚拟机在网络上显示为另一个物理设备。
  # config.vm.network "public_network"

  # 创建共享文件夹，第一个参数是本机路径，第二个是虚拟机路径
  # config.vm.synced_folder "../data", "/vagrant_data"

  # 禁用当前代码目录的默认共享。这样做可确保 vagrant box 无法访问Vagrantfile
  # 从而使 vagrant box 和主机之间隔离 。
  # config.vm.synced_folder ".", "/vagrant", disabled: true

  # 特定虚拟机提供商的配置
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #

  # 使用 shell 脚本启用配置。其他配置程序（例如 # Ansible、Chef、Docker、Puppet 和 Salt）也可用。
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end

```

当执行`vagrant up`时，vagrant会按照`Vagrantfile`的配置来依次执行：

+ 创建一个box
+ 配置虚拟网络
+ mount文件配置
+ 虚拟机平台配置
+ provision启动脚本

`box`就好比是基础镜像文件，类比dockerfile文件中的基础镜像。常见的box操作CLI：

```bash
vagrant box list
vagrant box add hashicorp/bionic64
vagrant box remove hashicorp/bionic64
```

指定box的版本和URL：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/bionic64"
  config.vm.box_url = "https://vagrantcloud.com/hashicorp/bionic64"
  config.vm.box_version = "1.0.282"
end

```

更多开箱即用的盒子在 [https://portal.cloud.hashicorp.com/vagrant/discover](https://portal.cloud.hashicorp.com/vagrant/discover)，类似dockerhub。

创建自己的 box 参考：
+ [https://developer.hashicorp.com/vagrant/docs/providers/virtualbox/boxes#creating-a-base-box](https://developer.hashicorp.com/vagrant/docs/providers/virtualbox/boxes#creating-a-base-box)
+ [https://developer.hashicorp.com/vagrant/vagrant-cloud/boxes/create](https://developer.hashicorp.com/vagrant/vagrant-cloud/boxes/create)
