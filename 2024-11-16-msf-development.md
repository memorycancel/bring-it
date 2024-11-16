---
title: 在 Ubuntu 上搭建 Metasploit-Framework 开发环境
layout: home
---

# 在 Ubuntu 上搭建 Metasploit-Framework 开发环境

2024-11-16 13:00

Metasploit-Framework 是一款渗透测试开源工具集合。
主要使用方式是 msfconsole 里面涵盖非常全面且热门的 CLI。

开源部分并没有提供 GUI，成熟的GUI由商业公司 rapid7 发布。
那么遵循免费最好，能用就行的原则。如何进行使用呢？

先按照官方文档进行安装：

[https://docs.metasploit.com/docs/development/get-started/setting-up-a-metasploit-development-environment.html](https://docs.metasploit.com/docs/development/get-started/setting-up-a-metasploit-development-environment.html)

尽量使用 Linux环境。
安装基础依赖：

```
sudo apt update && sudo apt install -y git autoconf build-essential libpcap-dev libpq-dev zlib1g-dev libsqlite3-dev
```

拉取源代码：

```
export GITHUB_USERNAME=YOUR_USERNAME_FOR_GITHUB
export GITHUB_EMAIL=YOUR_EMAIL_ADDRESS_FOR_GITHUB
mkdir -p ~/git
cd ~/git
git clone git@github.com:$GITHUB_USERNAME/metasploit-framework
cd ~/git/metasploit-framework
```

安装Ruby，此处使用 mise 安装。参考[使用 Omakese 在 Ubuntu 上搭建Ruby on Rails开发环境](2024-11-13-rails-dev-ubuntu)

安装gems包

```
cd ~/git/metasploit-framework/
gem install bundler
bundle install
```

然后就是有意思的地方，这个工具是自带数据库的，可以发现源代码中有 schema 文件。数据库使用的
是Postgresql。

安装Postfresql参考官方文档： [https://neon.tech/postgresql/postgresql-getting-started](https://neon.tech/postgresql/postgresql-getting-started)

修改`/etc/postgresql/16/main/postgresql.conf` 默认端口为 `5433`。
执行`systemctl restart postgresql`。

重要的是安装好后记得设置密码和端口等元信息，在配置msf的数据库链接时要用。

```
cd ~/git/metasploit-framework/
./msfdb init --connection-string="postgres://postgres:mysecretpassword@127.0.0.1:5433/postgres"
```
如果不带参数，msf会在 `~/.msf4` 文件夹创建默认的文件。在这里不要使用默认指令。
创建数据库链接[https://docs.rapid7.com/metasploit/managing-the-database/](https://docs.rapid7.com/metasploit/managing-the-database/)：
```
./msfconsole
db_status
db_connect your_msfdb_user:your_msfdb_pswd@127.0.0.1:5432/msf_database
db_status
```
![1](assets/images/2024-11-16/1.png)

那么如何使用msf渗透测试学习呢？msf提供了一个“练兵场”： [https://github.com/memorycancel/metasploitable3](https://github.com/memorycancel/metasploitable3)

尝试了很多安装方法，都没成功，最后找到了现成的镜像：[https://sourceforge.net/projects/metasploitable3-ub1404upgraded/files/](https://sourceforge.net/projects/metasploitable3-ub1404upgraded/files)
需要先安装好VirtualBox然后导入镜像，网络配置参考：[https://www.youtube.com/watch?v=QDgFry01uCA](https://www.youtube.com/watch?v=QDgFry01uCA)

![2](assets/images/2024-11-16/2.png)

启动并登陆机器，`vagrant:vagrant`:

![3](assets/images/2024-11-16/3.png)

机器的ip为：`172.28.128.3` ,进如 `msfconsole` 扫描他：

```
db_nmap 172.28.128.3
```
![4](assets/images/2024-11-16/4.png)

可以从 msfconsole 中通过输入model名看到数据。

![5](assets/images/2024-11-16/5.png)

也可以在数据库中看到。

![6](assets/images/2024-11-16/6.png)

