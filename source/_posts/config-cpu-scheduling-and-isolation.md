---
title: 配置cpu调度与隔离
date: 2024-11-21 20:50:23
tags:
  - hadoop
categories: 环境配置
---
在hadoop集里，你可以配置cpu调度, 为应用程序容器分配具有所需 CPU 资源的最佳节点。

1. 配置资源计算器

    在ResourceManager和NodeManager节点，将capacity-scheduler.xml中的`DefaultResourceCalculater`修改为`DominantResourceCalculator`以启用cup调用。

    * *Property:* `yarn.scheduler.capacity.resource-calculator`
    * *Value:* `org.apache.hadoop.yarn.util.resource.DominantResourceCalculator`
    * *Example:*

    ```xml
    <property>
      <name>yarn.scheduler.capacity.resource-calculator</name>
      <!-- <value>org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator</value> -->
      <value>org.apache.hadoop.yarn.util.resource.DominantResourceCalculator</value>
    </property>
    ```

2. 配置cpu核数

    在ResourceManager和NodeManager节点，将yarn-site.xml中的`cpu-vcores`修改为所需要的核数, 一般为NodeManager节点的物理核数。
    * *Property:* `yarn.nodemanager.resource.cpu-vcores`
    * *Value:* `<number_of_physical_cores>`
    * *Example:*

    ```xml
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>16</value>
    </property>
    ```

    也可以通过开启硬件自动检测来配置cpu核数。
    * *Property:* `yarn.nodemanager.resource.detect-hardware-capabilities`
    * *Value:* `true`
    * *Example:*

    ```xml
    <property>
        <name>yarn.nodemanager.resource.detect-hardware-capabilities</name>
        <value>true</value>
    </property>

    ```

    以上两个配置项，只需要配置其中一项就可以了。

3. 配置cpu隔离

    启用`cgroups`。`cgroups`是`CPU`进程的隔离机制。如果不激活`cgroups`，`DRF`调度程序会尝试平衡负载，这可能会出现不可预测的行为。
    启用`cgroups`, 请查看我的上一篇[blog](/2024/11/20/enable-cgroup2)。
