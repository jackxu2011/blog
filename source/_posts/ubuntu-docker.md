---
title: ubuntu-docker
date: 2025-04-02 15:21:47
tags:
  - docker
  - nvidia
  - ubuntu 24.04
categories: 软件安装
---
Ubuntu 24.04 下安装docker与nvidia-container-toolkit的过程。

## 一、安装docker

### 卸载旧版本docker

在安装docker引擎前，需要卸载所有冲突的包。操作系统可能已经安装了非官方的包，如：

* docker.io
* docker-compose
* docker-compose-v2
* docker-doc
* podman-docker

卸载命令如下：

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

`/var/lib/docker/`文件下的Images, containers, volumes, 和 networks不会自动删除，这些数据需要手动清除。

### 安装docker

本文中，我们使用`apt`源安装docker

1. 配置`apt`源安装

    ```bash
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
      $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    ```

2. 安装docker包

    ```bash
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

3. 配置docker镜像
    中国很多docker镜像都无法使用，目前1ms.run据说提供的是经过审查的docker镜像，可以正常使用，配置镜像的命令如下：

    ```bash
    curl -sSLO https://static.1ms.run/1ms-helper.bin && chmod +x 1ms-helper.bin && sudo ./1ms-helper.bin config:mirror && sudo systemctl daemon-reload && sudo systemctl restart docker
    ```

4. 将当前用户加入到docker组
    如果需要直接运行docker，需要将当前用户加到docker组里。命令如下`sudo usermod -a -G docker $USER`
    然后**退出重新打开终端**以当前配置生效

5. 测试docker是否安装成功

    ```bash
    docker run hello-world
    ```

6. (可选) 配置docker的数据目录
    docker的默认数据目录为`/var/lib/docker`, 如果需要修改数据目录.可以修改`/etc/docker/daemon.json`,
    假如目标地址为`/data/docker`, 则增加内容`"data-root": "/data/docker"`如下所示：

    ```json
    {
        "registry-mirrors": [
            "https://docker.1ms.run"
        ],
        "data-root": "/data/docker"
    }
    ```

    然后执行`sudo systemctl daemon-reload && sudo systemctl restart docker`使配置生效

## 二、安装nvidia-container-toolkit

本文中，我们使用`apt`源安装nvidia-container-toolkit

1. 配置`apt`源安装

    ```bash
    curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list && sudo apt-get update
    ```

2. 安装nvidia-container-toolkit

    ```bash
    sudo apt-get install -y nvidia-container-toolkit
    ```

## 总结

经过这些配置，一是解决了docker镜像问题，不过一些特殊的镜像还是得想其它的办法，二是一些需要nvidia卡的docker也可以正常拉起。
