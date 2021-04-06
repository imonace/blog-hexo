---
title: openSUSE MicroOS 做云容器 Host 的简单配置与使用经验
date: 2021-04-06 21:34:51
tags: [Linux]
---

最近想试用 openSUSE 家的 MicroOS 做容器小鸡的系统，主要看上了它无人值守自动更新并且方便回滚的特性。但无奈不管内网还是外网资料都很少，最终是在虚拟机里试错了几次后成功部署到了 VPS 上，也分享一下部署和使用的经验。

#### openSUSE MicroOS

openSUSE MicroOS 是一个基于 openSUSE Tumbleweed 的衍生发行版，[Wiki](https://en.opensuse.org/Portal:MicroOS) 中对其特点的介绍有：

- 小巧：旨在针对特定场景进行部署的轻量级映像
- 可扩展：针对大型部署优化，同时也可以作为单机操作系统
- 保持最新：自动应用更新，且不会影响正在运行的系统
- 快速恢复：发生故障时，系统会自动回滚到上一个工作状态
- 敏捷：不包含不必要的软件包

总结来说，openSUSE MicroOS 是一个“无需为它担心”的操作系统。定位比较像 Fedora CoreOS 和 Silverblue 的结合，也是openSUSE Kubic 的环境基础，与 Kubic 相比主要去掉了 Kubernetes 的相关工具，应用场景更贴近单点部署。

MicroOS 的一个重要特点是“事务式更新”，事务式更新介绍可见 [AstroProfundis 大佬的 Kubic 集群部署](https://forum.suse.org.cn/t/topic/13482)。

<!-- more -->

#### 架构

MicroOS 详细的架构设计可见 [Portal:MicroOS/Design](https://en.opensuse.org/Portal:MicroOS/Design) ，这里简单介绍一些日常使用时需要注意的点：

- 根目录使用 read-only 只读挂载，在运行中无法直接对系统进行修改。MicroOS 系统提供了一个叫 `transactional-update` 的工具，实际上是snapper等系统工具的封装。调用时会自动创建一个可写快照，在内部完成对系统的修改并退出后，系统会自动返回到创建快照之前，实际修改只有重启系统后才会生效。
- `transactional-update` 的一些常用指令有，`dup` 升级系统； `pkg` 安装、更新、卸载软件包； `shell` 进入可读写快照，手动完成系统修改； `cleanup` 清理不需要的快照。其余指令可见 [man-page](https://kubic.opensuse.org/documentation/man-pages/transactional-update.8.html) 。
- 用户目录、一些常规数据目录是可读写挂载，包括 /usr/local, /root, /home, /var, /opt ，这些目录都可以正常使用，且快照不包含这些目录。
- 配置文件目录 /etc 通过 overlayfs 挂载，实际文件保存在 /var/lib/overlay 中，也是可读写挂载。/etc 与普通目录的区别在于，普通目录在后一个快照运行时的修改，在返回前一个快照时也可见，而通过 overlayfs 挂载的 /etc 可以通过快照返回上一个正常的节点。简单来说，就是为了让配置文件可以直接修改而不用重启的同时，防止配置文件修改把系统“滚挂”。

#### 初始化安装

首先将 MicroOS 的安装 ISO 挂载到云服务商的虚拟主机上，MicroOS 提供了 4 种默认安装配置，分别是预装和不预装容器环境的服务器配置，和使用 Gnome 或 KDE 的桌面配置。这篇主要介绍 MicroOS 在云服务器上的使用，不安装桌面环境。

预装容器环境的默认容器管理程序是 `podman` ，与 `docker` 的主要区别是没有守护进程。如果想使用 `docker` 可以先选择不预装容器环境的安装配置，进入系统后再手动安装 `docker` 。此外安装时还可以选择网络管理程序 `wicked` 或 `NetworkManager` ，`docker` 建议搭配 `NetworkManager`，`podman` 建议搭配 `wicked` 。

同时安装时建议直接取消安装 SELinux ，见下。

#### *关闭SELinux

也不知道为什么一般都用 AppArmor 的 oS 会在 MicroOS 上默认安装 SELinux。。SELinux 的复杂配置我搞不懂，我在配置系统时深受其害，一般一个正常系统不开 SELinux 也具有足够的安全性。

下面是取消安装或禁用 SELinux 的方法：

##### 安装时取消安装

1. 在安装程序最后的 Install Summary 中，将 Security 子项里的 SELinux Default Mode 设为 Disabled
2. 在 Software 子项里取消勾选 SELinux Support

##### 已经安装后禁用

1. 编辑 /etc/default/grub ，将 GRUB_CMDLINE_LINUX_DEFAULT 行尾的 selinux=1 改写为 0
2. 执行 `transactional-update grub.cfg` ，重启

#### 常用基础配置

接下来是一些 oS 做服务器的简单基础配置，有经验的朋友就可以跳过了。

##### 常用软件

安装软件可以用 `transactional-update pkg in` ，也可以进 `transactional-update shell` 中操作，我还是习惯先进 shell。

```shell
zypper in git tmux htop vim # 常用软件按需安装，默认的vim是vim-small，没高亮
zypper in fail2ban nginx bash-completion
zypper in docker python3-docker-compose # 安装 docker（ 如果没有安装 podman ）
```

安装完成后 `exit` 退出，`systemctl reboot` 重启即可应用修改。

##### 添加普通用户并授予 sudo 权限

```shell
groupadd -r wheel # 默认没有wheel组
useradd -m -u 1000 -G wheel -s /bin/bash 用户名
passwd 用户名    # 修改新用户密码
```

此时已经可以用新用户登陆了，不过 sudo 会请求 root 的密码，再修改让 sudo 请求用户密码。

```shell
EDITOR=vim visudo
注释掉 "Defaults targetpw" 和 "ALL   ALL=(ALL) ALL" 两行
取消注释 "%wheel ALL=(ALL) ALL"
```

##### 容器环境

如果安装的是 `docker` ，直接用 `systemctl enable --now docker.service` 即可。

如果安装的是 `podman` ，可以用内建的 `podman generate systemd` 命令为容器创建守护服务。

以 subconverter 容器为例：

```bash
podman pull tindy2013/subconverter
podman run -d --name=subconverter -p 25500:25500 tindy2013/subconverter
podman generate systemd -n subconverter | tee /etc/systemd/system/subconverter.service
systemctl enable subconverter.service
```

Happy hacking ~
