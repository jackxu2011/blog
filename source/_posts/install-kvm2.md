---
title: 在Ubuntu22.04中安装KVM(二)
date: 2024-01-14 13:16:53
tags:
  - kvm
  - ubuntu 22.04
  - linux
  - webvirtcloud
categories: 软件安装
---
WebVirtCloud 是一个基于 Web 的 KVM 虚拟化管理工具。它允许管理员和用户从 Web 界面创建、管理和删除在 KVM 管理程序上运行的虚拟机。它基于 Django 构建，支持基于用户的授权和身份验证。使用 WebVirtCloud，您可以从单个安装管理多个  QEMU/KVM Hypervisor、管理 Hypervisor 网络和管理数据存储池。

因为在实体机上直接安装WebVirtCloud, 会出现比较多的问题，比如我已经安装了conda，但是安装手册里又要安装venv等问题。 所以这里选用docker进行webvirtcloud的安装。

这个可以直接通过webVirtCloud的wiki查看到[https://github.com/retspen/webvirtcloud/wiki/Docker-Installation-&-Update](https://github.com/retspen/webvirtcloud/wiki/Docker-Installation-&-Update), 这个链接有时可能需要科学上网。

也可以参考我同步的[gitee仓库](https://gitee.com/kalina/webvirtcloud/wikis/Docker-Installation-&-Update)

我这里对几处地方进行修改，以方便大家参考。

## 安装步骤

首先需要下载webvirtcloud, 可以webVirtCloud的wiki上的1-3步。

### 1. 修改Dockerfile, 使其支持时区设置，解决增加计算节点的出昏问题，python依赖问题与数据文件本地化的问题

* 增加tzdata的安装， 与环境变量 ```TZ="Asia/Shanghai"``` 解决时区问题
* 增加ssh-askpass的安装, 并新建 ```mkdir /var/www/.ssh``` 解决增加计算节点出错问题
* 增加```pip3 config set``` 配置以解决python环境安装，无法下载依赖问题
* 增加```mkdir db``` 以使sqlLite数据文件可以本地化，以防止docker更新导致数据丢失，同时这个需要配置修改setting.py文件

以下是通过git diff 命令查看到的修改记录：

```shell
diff --git a/Dockerfile b/Dockerfile
index f86f479..c2ea76a 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -12,6 +12,7 @@ RUN echo 'APT::Get::Clean=always;' >> /etc/apt/apt.conf.d/99AutomaticClean
 RUN apt-get update -qqy \
     && DEBIAN_FRONTEND=noninteractive apt-get -qyy install \
        --no-install-recommends \
+       tzdata \
        git \
        python3-venv \
        python3-dev \
@@ -25,15 +26,22 @@ RUN apt-get update -qqy \
        libssl-dev \
        libsasl2-dev \
        libsasl2-modules \
+        ssh-askpass \
     && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

+ENV TZ="Asia/Shanghai"
+
 COPY . /srv/webvirtcloud
 RUN chown -R www-data:www-data /srv/webvirtcloud

+
 # Setup webvirtcloud
 WORKDIR /srv/webvirtcloud
+RUN mkdir db
 RUN python3 -m venv venv && \
        . venv/bin/activate && \
+       pip3 config set global.index-url http://mirrors.aliyun.com/pypi/simple/ &&\
+       pip3 config set install.trusted-host mirrors.aliyun.com &&\
        pip3 install -U pip && \
        pip3 install wheel && \
        pip3 install -r conf/requirements.txt && \
@@ -66,4 +74,7 @@ COPY conf/runit/webvirtcloud.sh               /etc/service/webvirtcloud/run
 # Define mountable directories.
 #VOLUME []

+RUN mkdir /var/www/.ssh
+RUN chown -R www-data:www-data /var/www
+
 WORKDIR /srv/webvirtcloud
 ```

### 2. 修改webvirtcloud/settings.py.template中的数据库目录，以方便将数据文件进行挂载

* 增加 DATA_DIR 变量
* 修改 DATABASE_ 位置

以下是通过git diff 命令查看到的修改记录：

```shell
diff --git a/webvirtcloud/settings.py.template b/webvirtcloud/settings.py.template
index de07701..b394be8 100644
--- a/webvirtcloud/settings.py.template
+++ b/webvirtcloud/settings.py.template
@@ -98,10 +98,13 @@ WSGI_APPLICATION = "webvirtcloud.wsgi.application"
 # Database
 # https://docs.djangoproject.com/en/3.0/ref/settings/#databases

+DATA_DIR = "db"
+DATA_ROOT = Path.joinpath(BASE_DIR, DATA_DIR)
+
 DATABASES = {
     "default": {
         "ENGINE": "django.db.backends.sqlite3",
-        "NAME": Path.joinpath(BASE_DIR, "db.sqlite3"),
+        "NAME": Path.joinpath(DATA_ROOT, "db.sqlite3"),
     }
 }
```

### 4. 新建本地文件夹，并生成key

命令如下：

```shell
## 创建文件夹
sudo mkdir -p /data/webvirtcloud/ssh
sudo mkdir -p /data/webvirtcloud/db
## 生成key
sudo ssh-keygen -t rsa -N '' -f /data/webvirtcloud/ssh/id_rsa -q
## 设置ssh配置文件，方便免密登录
sudo touch /data/webvirtcloud/ssh/config
echo -e "StrictHostKeyChecking=no\nUserKnownHostsFile=/dev/null" | sudo tee -a /data/webvirtcloud/ssh/config
## 设置权限，因为容器内部www-data的gid,uid为33，所以这里直接设置为33

sudo chown 33:33 -R /data/webvirtcloud
sudo chmod 700 /data/webvirtcloud/ssh
sudo chmod 600 /data/webvirtcloud/ssh/*
```

**拉下来-可以继续执行webVirtCloud的wiki上的4-7步。**

### 5. 开启debug模式，最新版本在非debug模式下无法新建虚拟机

```shell
sed -i 's/DEBUG.*/DEBUG = True/g' webvirtcloud/settings.py
```

### 6. 修改host配置

```shell
## 192.168.1.162 为你本机的ip地址
sed -i 's/localhost/192.168.1.162/g' webvirtcloud/settings.py
```

### 7. 增加docker-compose.yaml以方便用docker方式部署

文件内容如下：

```yaml
version: "3"

services:
  wvc:
    image: retspen/webvirtcloud:1
    ports:
      - "80:80"
      - "6080:6080"
    environment:
     - TZ=Asia/Shanghai
    volumes:
      - ./webvirtcloud/settings.py:/srv/webvirtcloud/webvirtcloud/settings.py
      - /data/webvirtcloud/ssh:/var/www/.ssh
      - /data/webvirtcloud/db:/srv/webvirtcloud/db
```

### 8. 启动webvirtcloud

```shell
docker compose up -d
```

**注意安装过程中不要随便切换当前目录，即webvirtcloud的检出目录。**
