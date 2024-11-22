---
title: flink 的 yarn-session 命令说明
date: 2024-11-22 16:09:28
tags:
  - flink
categories: 大数据
---

为什么写这一个说明呢？因为网上关于yarn-session命令都是错误的，导致集群无法启动，或者命令行参数不生效。

## 1. 网上常见命令

  ```bash
  bin/yarn-session.sh -n 2 -jm 1024 -tm 1024 -d
  ```

  这个命令会启动一个`flink`的`yarn-session`集群，但是后面的参数都不生效(flink-1.15及以上， 本人只查看这个版本以后的代码。其它版本能否使用不清楚)。

  这个原因是：`flink`的`FlinkYarnSessionCli`中对与未知参数的处理使用`stopAtNonOptions=true`的选项，导致后面的参数都失效， 而不会抛出`Unrecognized option:`异常。

## 2. 正确命令

通过`bin/yarn-session.sh -h` 命令查看参数说明，可以发现这个命令中并没有-n这个参数，见下图。

```bash
Usage:
   Optional
     -at,--applicationType <arg>     Set a custom application type for the application on YARN
     -D <property=value>             use value for given property
     -d,--detached                   If present, runs the job in detached mode
     -h,--help                       Help for the Yarn session CLI.
     -id,--applicationId <arg>       Attach to running YARN session
     -j,--jar <arg>                  Path to Flink jar file
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container with optional unit (default: MB)
     -m,--jobmanager <arg>           Set to yarn-cluster to use YARN execution mode.
     -nl,--nodeLabel <arg>           Specify YARN node label for the YARN application
     -nm,--name <arg>                Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -t,--ship <arg>                 Ship files in the specified directory (t for transfer)
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container with optional unit (default: MB)
     -yd,--yarndetached              If present, runs the job in detached mode (deprecated; use non-YARN specific option instead)
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for high availability mode
```

从上面输出的使用参数说明中，可以发现，`-n`参数已经不存在。 而且所有的参数都是可选参数， 这里对参数进行说明：

- `-at,--applicationType <参数>`：为在YARN上运行的应用程序设置一个自定义的应用程序类型。这有助于分类和监控。
- `-D <属性=值>`：为Flink作业或YARN环境设置配置属性。可以通过多次使用此标志来设置多个属性。
- `-d,--detached`：如果使用此标志，作业将以分离模式运行，即提交作业后命令提示符会立即返回，而不是等待作业完成。
- `-h,--help`：显示YARN会话CLI的帮助信息，如果您忘记了语法或需要了解可用选项的更多信息，这非常有用。
- `-id,--applicationId <参数>`：使用提供的应用程序ID附加到正在运行的YARN会话。当您希望与已经运行的应用程序交互时，这很有用。
- `-j,--jar <参数>`：指定包含要执行作业的Flink jar文件的路径。
- `-jm,--jobManagerMemory <参数>`：设置分配给JobManager容器的内存量，可选单位（例如MB、GB）。默认单位是MB。
- `-m,--jobmanager <参数>`：通过将参数设置为`yarn-cluster`来指示Flink使用YARN执行模式。
- `-nl,--nodeLabel <参数>`：为应用程序指定YARN节点标签，这可以帮助基于节点标签进行资源分配。
- `-nm,--name <参数>`：为YARN上的应用程序设置自定义名称，这可以使在YARN ResourceManager UI中识别应用程序变得更加容易。
- `-q,--query`：显示YARN集群上的可用资源，如内存和CPU核心数，这对于计划资源分配非常有用。
- `-qu,--queue <参数>`：指定应提交应用程序的YARN队列。队列用于跨不同团队或项目管理资源分配。
- `-s,--slots <参数>`：确定每个TaskManager的槽位数量。槽位表示可以由TaskManager同时执行的并行任务。
- `-t,--ship <参数>`：在启动作业之前，从本地文件系统将文件传输到YARN集群节点。这对于传输作业运行时可能需要的额外文件非常有用。
- `-tm,--taskManagerMemory <参数>`：设置分配给每个TaskManager容器的内存量，可选单位（例如MB、GB）。默认单位是MB。
- `-yd,--yarndetached`：类似于`-d`或`--detached`标志，但已废弃，建议使用非YARN特定的选项。
- `-z,--zookeeperNamespace <参数>`：指定用于高可用性（HA）模式的ZooKeeper子路径创建的命名空间。仅在使用ZooKeeper进行HA配置时需要。

所以yarn-session.sh命令常用格式如下：

```bash
./bin/yarn-session.sh -nm "mySession" -jm 1024 -tm 2048 -s 2 -d
```

此命令告诉YARN启动一个具有1GB内存的JobManager、每个TaskManager有2GB内存和两个槽位的Flink集群，所有这些都在名为“mySession”的应用程序下。

至此，了解到为什么yarn-session.sh命令参数不生效的原因; 建议flink将`stopAtNonOptions=true`改为`stopAtNonOptions=false`
