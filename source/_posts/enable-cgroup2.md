---
title: 怎么启用cgroup2
date: 2024-11-20 10:14:52
tags:
  - linux
categories: 环境配置
---
本文介绍如何查看系统是否支持cgroup2，是否启用，以及如何启用cgroup2。

## 1. 查看系统是否支持cgroup2

```bash
grep cgroup /proc/filesystems
```

输出包含`nodev cgroup2`，则表明系统支持cgroup2。

## 2. 查看系统是否启用cgroup2

```bash
state -fc %T /sys/fs/cgroup
```

如果输出为`tmpfs`，则表明系统启用cgroup v1, 没有启用cgroup v2。
如果输出为`cgroup2fs`，则表明系统启用cgroup2。

## 3. 启用cgroup2

* 在`/etc/default/grub`中的`GRUB_CMDLINE_LINUX=`添加`systemd.unified_cgroup_hierarchy=1`
* 修改完成后以root用户执行`update-grub`或`sudo update-grub`, 如果是ubuntu，则执行`sudo grub2-mkconfig`
* 执行`apt/yum upgrade`
* 重启系统并查看cgroup2是否启用
