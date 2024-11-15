---
title: mac生成新的SSH密钥并添加到ssh-agent
date: 2024-05-24 14:06:09
tags:
  - git
  - ssh
  - mac
categories: 环境配置
---

可以使用SSH来访问Git服务与你的运程主机。通过SSH，你可以避免每次使用密码，而直使用本地计算机上的私钥文件进行身份验证。
有关详细信息，请阅读[什么是SSH](https://info.support.huawei.com/info-finder/encyclopedia/zh/SSH.html)。

由于一些安全性低的算法已经被弃用，所以这篇文章记录一下新的配置步骤。

## 生成新的SSH密钥

1. 打开终端，输入`ssh-keygen -t ed25519 -C "your_email@example.com"`
    > 如果你系统不支持ed25519，可以使用`ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`

    这将以提供的电子邮件地址创建新的SSH密钥。

    ```bash
    > Generating public/private ALGORITHM key pair.
    ```

    当系统提示“Enter a file in which to save the key”时，按回车键以使用默认值。请注意，如果以前创建了 SSH 密钥，则 ssh-keygen 可能会要求重写另一个密钥，在这种情况下，我们建议创建自定义命名的 SSH 密钥。 为此，请键入默认文件位置，并将 id_ALGORITHM 替换为自定义密钥名称。

2. 输入一个密码，然后确认密码。

    ```bash
    > Enter passphrase (empty for no passphrase): [Type a passphrase]
    > Enter same passphrase again: [Type passphrase again]
    ```

    > 如果你不想输入密码，可以按回车键。


## 将SSH密钥添加到ssh-agent

1. 启动ssh-agent。

    ```bash
    $ eval "$(ssh-agent -s)"
    > Agent pid 59566
    ```

2. 修改配置文件`~/.ssh/config`，以自动将私钥添加到ssh-agent中并在密钥链中保存密码。

    以github为例，添加以下内容：

    ```text
    Host github.com
      IdentityFile ~/.ssh/id_ed25519
      AddKeysToAgent yes
      UseKeychain yes
    ```

3. 将私钥添加到ssh-agent并将密码保存到密钥链。

    ```bash
    ssh-add --apple-use-keychain ~/.ssh/id_ed25519
    ```

4. 将SSH公钥添加到[GitHub](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)，[gitlab](https://jaminzhang.github.io/git/config-and-add-SSH-key-in-GitLab/)或你相要访问的主机`ssh-copy-id -i ~/.ssh/mykey user@host`

5. 设置开机自启以zsh为例

在 ~/.zprofile 增加以下配置, 根据你需要设置的key自行修改start_ssh_agent 函数

```bash
# 自动运行 ssh-agent 并添加 SSH 密钥
function start_ssh_agent {
    if [ -z "$SSH_AUTH_SOCK" ]; then
        echo "Starting ssh-agent..."
        eval $(ssh-agent -s)
        ssh-add ~/.ssh/id_ed25519
    else
        echo "ssh-agent already running."
    fi
}

# 在 zsh 启动时调用 start_ssh_agent
start_ssh_agent
```
