---
title: java运程调试
date: 2024-06-02 11:48:40
tags:
  - java
  - debug
categories:
  - java
---

远程调试Java程序是一种在开发环境以外的服务器或设备上运行的Java应用进行调试的方法。这对于无法直接在IDE中运行和调试的应用尤其有用。以下是使用Java Remote Debugging的一般步骤：

## 1. 准备应用启动参数

配置远程调试端口：在启动Java应用时，需要添加特定的JVM参数来启用远程调试模式。基本的参数格式如下：

* jdk 1.5 - 1.8:

```sh
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=port
```

* jdk 1.9+:

```sh
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:port
```

transport=dt_socket 指定使用套接字通信。
server=y 表示该Java进程作为调试服务器等待连接。
suspend=n 控制Java应用是否在启动时暂停，直到调试器连接 (n 表示不暂停，直接启动；y 表示暂停)。
address=*:port 指定监听的地址和端口号，* 表示监听所有IP地址(jdk1.9以下没有这个参数)，port 是你选择的端口号，例如 5005。

## 2. 启动Java应用

启动应用：使用上面配置的参数启动你的Java应用。例如，如果你的应用通常使用 java -jar yourapp.jar 启动，那么现在应该改为：

```sh
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -jar yourapp.jar
```

## 3. 配置IDE进行远程调试

IntelliJ IDEA:

1. 打开菜单 Run > Edit Configurations...
2. 点击 + 添加新的配置，选择 Remote 或 Remote JVM Debug
3. 设置 Host 为你应用所在的IP地址，Port 为你之前指定的端口号（如5005）
4. 点击 OK 保存配置，然后选择这个配置并点击运行按钮旁边的调试图标开始调试。

## 4. 连接到远程应用

开始调试：在IDE中启动刚才创建的远程调试配置。IDE会尝试连接到你在目标机器上配置的端口。
一旦连接成功，你就可以像本地调试一样设置断点、查看变量值、单步执行代码等。
确保网络配置允许从你的开发机器访问远程服务器的调试端口，并且没有防火墙阻止此类连接。
此外，如果远程服务器使用的是非标准的JDK，确保IDE中配置的JDK兼容性设置正确。
