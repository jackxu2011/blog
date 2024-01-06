---
title: 在Ubuntu22.04中安装KVM(一)
date: 2024-01-05 13:12:59
tags:
  - kvm
  - ubuntu 22.04
  - linux
categories: 软件安装
---
KVM(Kernel-based Virtual Machine)是基于内核的虚拟机的首字母缩写, 这是一项集成在内核中的开源虚拟化技术。虚拟机是一种软件应用程序，可作为另一台实体计算机中的独立计算机使用。虚拟机与实体计算机共享 CPU 周期、网络带宽和内存等资源。KVM 是 Linux 操作系统组件，它为 Linux 上的虚拟机提供原生支持。自 2007 年以来，它已在 Linux 发行版中推出。

本文将介绍在Ubuntu22.04(jammy) 中如何安装KVM。

## 一、 安装准备

在安装KVM前需要将Ubuntu的软件进行更新，并检查本系统是否支虚拟化，同时检查本系统是否能够运行KVM虚拟机。

### 1. 系统更新

```bash
sudo apt update
sudo apt upgrade
```

### 2. 检查本系统是否支虚拟化

检查CPU是否支持KVM虚拟化，如英特尔处理器的VT-x(vmx)或AMD处理器的AMD-V(svm)虚拟化技术。
可以通过运行如下命令，如果输出值大于 0，那么虚拟化被启用。如果虚拟化功能没有启用，请确保在系统的BIOS设置中启用虚拟化功能;同时确保CPU支持虚拟化技术。

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

### 3. 检查本系统是否能够运行KVM虚拟机

* 安装cpu-checker

```bash
sudo apt install -y cpu-checker
```

* 执行kvm-ok命令

```bash
kvm-ok
```

如果支持运行KVM虚拟机将会有如下输出

```text
INFO: /dev/kvm exists
KVM acceleration can be used
```

如果以上检查都通过，可以进入下一步进行KVM安装。

## 二、安装KVM

安装KVM主要分为以下几步：

1. 安装KVM软件
2. 启动虚拟化守护进程
3. 配置网络

### 1. 安装KVM软件

通过如下命令在 Ubuntu 22.04 中安装 KVM 以及其他相关虚拟化软件包：

```bash
sudo apt install -y qemu-kvm libvirt-daemon-system virtinst libvirt-clients bridge-utils virt-manager
```

* qemu-kvm: 一个提供硬件仿真的开源仿真器和虚拟化包
* libvirt-daemon-system: 为运行 libvirt 进程提供必要配置文件的工具
* virtinst: 一套为置备和修改虚拟机提供的命令行工具
* libvirt-clients: 一组客户端的库和API，用于从命令行管理和控制虚拟机和管理程序
* bridge-utils: 一套用于创建和管理桥接设备的工具
* virt-manager: 一款通过 libvirt 守护进程，基于 QT 的图形界面的虚拟机管理工具

virt-manager是cs结构的虚拟机管理工具，在下一篇中，将安装[webvirtcloud](https://github.com/retspen/webvirtcloud)来管理KVM虚拟机。

将当前登录用户加入 kvm 和 libvirt 用户组，以便能够创建和管理虚拟机。

```bash
sudo usermod -aG kvm $USER
sudo usermod -aG libvirt $USER
```

以上增加用户组的操作需要**重新登录**才能使得配置生效。

### 2. 启动虚拟化守护进程

在所有软件包安装完毕之后，通过如下命令启用并启动libvirt守护进程：

```bash
sudo systemctl enable --now libvirtd
sudo systemctl start libvirtd
```

可以通过以下命令查看libvirtd的运行状态:

```bash
sudo systemctl status libvirtd
```

如果服务正常将会看到类似信息：(包含：Active: active (running))

```text
● libvirtd.service - Virtualization daemon
     Loaded: loaded (/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2024-01-05 14:48:00 CST; 1 day 5h ago
TriggeredBy: ● libvirtd-admin.socket
             ● libvirtd-ro.socket
             ● libvirtd.socket
       Docs: man:libvirtd(8)
             https://libvirt.org
```

### 3. 配置网络

如果需要从外部访问 KVM 虚拟机，必须将虚拟机的网卡映射至网桥。virbr0 网桥是 KVM 安装完成后自动创建的，仅做测试用途。

可以通过如下内容在 /etc/netplan 目录下创建文件 01-netcfg.yaml 来新建网桥, 步骤如下：

* 1. 如果在/etc/netplan下已经有配置文件,如:00-installer-config.yaml，可以先备份。

```bash
cd /etc/netplan
mv 00-installer-config.yaml 00-installer-config.yaml.orig
```

* 2. 查看现有网卡信息

```bash
ip addr
```

返回信息如下：

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 08:00:27:4b:1d:45 brd ff:ff:ff:ff:ff:ff
    altname enp1s0f0
    inet 192.168.1.162/24 brd 192.168.90.255 scope global eno1
       valid_lft forever preferred_lft forever
    inet6 fe80::1a66:daff:fef5:6c98/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到第2条网卡信息显示：
  interface: eno1
  addresses(inet): 192.168.1.162/24
  macaddress(link/ether): 08:00:27:4b:1d:45

* 3. 配置01-netcfg.yaml

```bash
network:
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
  # add configuration for bridge interface
  bridges:
    br0:
      interfaces: [eno1]
      dhcp4: false
      addresses: [192.168.1.162/24]
      macaddress: 08:00:27:4b:1d:45
      routes:
        - to: default
          via: 192.168.1.1
          metric: 100
      nameservers:
        addresses: [192.168.1.1]
      dhcp6: false
  version: 2
```

保存并退出文件。

* 4. 通过netplan apply 命令应用上述变更

```bash
sudo netplan apply
```

可以通过如下 ip addr 命令，查看是否出现新的interface为br0的网卡，
并用查看macaddress与addresses是否与你配置的相同。
