---
title: 在Ubuntu22.04中安装cuda、cuDNN
date: 2024-03-15 16:52:12
tags:
  - cuda
  - cuDNN
  - ubuntu 22.04
categories: 软件安装
---
Ubuntu 22.04 下安装cuda、cuDNN的过程。

## 一、检查是否安装nvidia显卡

通过命令```lspci | grep -i nvidia``` 查看是否安装nvidia显卡，如果有输出则说明系统有nvidia显示。
输出如下：

```text
00:05.0 3D controller: NVIDIA Corporation GP102GL [Tesla P40] (rev a1)
```

## 二、安装NVIDIA显卡的官方驱动

1. 增加官方的apt源

    从[NVIDIA官网CUDA下载页面](https://developer.nvidia.com/cuda-downloads)获取apt源。
    选择Linux --> x86_64 --> Ubuntu --> 22.04 --> deb(network),
    具体的系统架构，根据真实系统进行选择。可以得到具体的安装命令。

2. 执行安装命令

    ```bash
    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
    sudo dpkg -i cuda-keyring_1.1-1_all.deb
    sudo apt-get update
    ```

3. 获取可用驱动程序

    ```bash
    ubuntu-drivers devices
    ```

    输出信息为：

    ```text
    == /sys/devices/pci0000:00/0000:00:02.0/0000:04:00.0 ==
    modalias : pci:v000010DEd00001B38sv000010DEsd000011D9bc03sc02i00
    vendor   : NVIDIA Corporation
    model    : GP102GL [Tesla P40]
    driver   : nvidia-driver-515 - third-party non-free
    driver   : nvidia-driver-545 - third-party non-free
    driver   : nvidia-driver-525 - third-party non-free
    driver   : nvidia-driver-535-server - distro non-free
    driver   : nvidia-driver-470 - distro non-free
    driver   : nvidia-driver-520 - third-party non-free
    driver   : nvidia-driver-525-server - distro non-free
    driver   : nvidia-driver-535 - third-party non-free
    driver   : nvidia-driver-550 - third-party non-free recommended
    driver   : nvidia-driver-470-server - distro non-free
    driver   : nvidia-driver-418-server - distro non-free
    driver   : nvidia-driver-450-server - distro non-free
    driver   : nvidia-driver-390 - distro non-free
    driver   : xserver-xorg-video-nouveau - distro free builtin
    ```

    找前面为driver，后面为recommend 的记录。
    上面的输出结果为：```nvidia-driver-550```

4. 安装驱动

    ```bash
    sudo apt install nvidia-driver-550
    ```

    安装完成后，重启电脑

5. 查看驱动是否安装成功

    ```bash
    nvidia-smi
    ```

    输出结果如下显示有一块显卡:

    ```text
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 550.54.14              Driver Version: 550.54.14      CUDA Version: 12.4     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  Tesla P40                      Off |   00000000:00:05.0 Off |                    0 |
    | N/A   28C    P8             11W /  250W |       0MiB /  23040MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+

    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+
    ```

## 三、安装cuda、cuDNN

可以参考[cuda官网](https://developer.nvidia.com/cuda-downloads)
与[cuDNN官网](https://developer.nvidia.com/cudnn-downloads)

下面是已经设置过官方源(**在本文第二部分已经设置过官方源**)后的执行过程：

1. 执行安装命令

    ```bash
    sudo apt-get -y install cuda-toolkit-12
    sudo apt-get -y install cudnn-cuda-12
    ```

2. 配置环境变量并验证

    bash环境下，通过命令```cat >> .bashrc << EOF```逐行输入以下文本

    zsh环境下，通过命令```cat >> .zshrc << EOF```逐行输入以下文本

    ```bash
    export CUDA_HOME=/usr/local/cuda
    export LD_LIBRARY_PATH=$CUDA_HOME/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
    export PATH=$CUDA_HOME/bin${PATH:+:${PATH}}
    EOF
    ```

    首次配置完环境变量后通过```source .bashrc```或```source .zshrc```命令使环境变量生效。

    配置完环境变量后，通过命令```nvcc -V```检查cuda版本，输出结果如下所示：

    ```text
    nvcc: NVIDIA (R) Cuda compiler driver
    Copyright (c) 2005-2024 NVIDIA Corporation
    Built on Tue_Feb_27_16:19:38_PST_2024
    Cuda compilation tools, release 12.4, V12.4.99
    Build cuda_12.4.r12.4/compiler.33961263_0
    ```

3. 验证与测试

    * 安装必要的依赖

    ```bash
    sudo apt-get install libfreeimage3 libfreeimage-dev
    ```

    * 编译测试代码

    ```bash
    cd ~
    cp -r /usr/src/cudnn_samples_v9 ./
    cd cudnn_samples_v9/mnistCUDNN
    make clean && make
    ```

    * 运行测试代码

    ```bash
    ./mnistCUDNN
    ```

    如果cudnn安装成功会显示如下信息：

    ```text
    Test passed!
    ```
