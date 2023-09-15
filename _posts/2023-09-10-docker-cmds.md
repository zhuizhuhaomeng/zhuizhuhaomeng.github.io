---
layout: post
title: "我常用的 docker 命令"
description: "我常用的 docker 命令"
date: 2023-09-09
tags: [docker]
---

# 让容器开机启动

首次执行使用这个命令

```shell
docker run --restart=always --name container-name -d 4e97feadb276 /bin/bash
```

修改已经存在的容器使用这个命令

```shell
docker update --restart=always 0576df221c0b
```

# 把容器当成虚拟机来执行

## 构建容器

把下面的配置保存到文件 rocky8.dockerfile

```dockerfile
FROM rockylinux:8.6

RUN yum makecache && yum install -y systemd

RUN printf "#!/bin/bash\n\nset -e -x\n\nexec /sbin/init\n" > /my-init

ENV container docker

CMD ["/bin/bash", "/my-init"]
```

执行以下命令创建容器

```shell
docker build -t rocky8-vm  - < rocky8.dockerfile
docker run --name rocky8-vm --restart=always --privileged -d rocky8-vm:latest
```

## jenkins 客户端作为服务启动

下面是 jenkins 客户端作作为服务运行的例子

### 创建目录启动脚本

```shell
mkdir -p /root/jenkins-agent/logs
mkdir -p /root/jenkins
cd /root/jenkins-agent
echo da8917dd102aeafd3399038048402683377 > secret-file
curl -sO http://192.168.50.201/jnlpJars/agent.jar
echo 'java -jar agent.jar -jnlpUrl http://192.168.50.201/manage/computer/rocky8%2Ddocker%2Dljl/jenkins-agent.jnlp -secret @secret-file -workDir "/root/jenkins"' > start.sh
chmod a+x start.sh
```

### service 配置文件

该配置文件保存在 /etc/sytemd/system

```
[Unit]
Description=Golang Application Demo
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
WorkingDirectory=/root/jenkins-agent
ExecStart=/bin/bash /root/jenkins-agent/start.sh
Restart=always
StandardOutput=append:/root/jenkins-agent/logs/access.log
StandardError=append:/root/jenkins-agent/logs/error.log

[Install]
WantedBy=multi-user.target
```

### 启动服务

```shell
systemctl enable jenkins-agent
systemctl start jenkins-agent
```
