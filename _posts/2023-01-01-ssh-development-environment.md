---
layout: post
title: "SSH 打造开发环境"
description: "SSH 打造开发环境"
date: 2023-01-01
feature_image: img/ssh/GFW.jpg
tags: [ssh]
---

# 使用 SSH 访问国际网络

```shell
# 如果防火墙打开的话
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
ssh -qTfnN -D 0.0.0.0:8080 user@remote-host
```

使用 SSH 构建一个 SOCK5 代理，我们使用到如下的 SSH 选项：

```text
  -q      Quiet mode.  Causes most warning and diagnostic messages to be suppressed
  -f      Requests ssh to go to background just before command execution.
  -T      Disable pseudo-terminal allocation.
  -n      Redirects stdin from /dev/null (actually, prevents reading from stdin).
  -N      Do not execute a remote command.
```

为了防止 SSH 断开，应该配置一下

# 办公室访问家庭机器

通过下面的命令把家庭的机器的 22 端口映射到公网的 9022 端口，这样就可以通过访问公网的 9022 端口来访问家庭的机器。

```shell
ssh -qfnNT -R  0.0.0.0:9022:127.0.0.1:22 user@remote-host
```

# 本机访问虚拟机里的 Web 服务

下面的命令将远程的 80 端口映射为本机的 8080 端口，这样就可以通过浏览器访问 8080 端口实现访问远程机器的目的。

```shell
ssh -qfnNT -L R 127.0.0.1:8080:127.0.0.1:80 user@remote-host
```


## ssh 保活配置

为了防止 ssh 连接远程机器的时候链接经常断开，我们需要给 ssh 的配置文件增加如下的报告配置。


```shell
mkdir ~/.ssh/
touch ~/.ssh/

cat >> ~/.ssh/config  << EOF
Host *
    ServerAliveInterval 60
EOF
chmod 600 ~/.ssh/config
```

# SSH 隧道图解

这张图来自 https://iximiuz.com/en/posts/ssh-tunnels/

![](../img/ssh/ssh-tunnels.png)


