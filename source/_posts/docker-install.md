---
title: centos7安装docker
date: 2017-11-22 20:39:10
categories:
- Docker
tags: 
- docker
- centos7
---
## 安装docker

```bash
#aliyun镜像 
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
#daocloud镜像 
curl -fsSL https://get.docker.com/ | sh
```
## 配置加速器

```bash
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["aliyun加速地址"]
}
EOF
```
<!-- more -->
## 修改允许非安全的仓库

```bash
vi /usr/lib/systemd/system/docker.service
```
找到ExecStart属性，在dockerd后面添加--insecure-registry 服务器IP:Docker仓库端口 ，最终为：

```bash
ExecStart=/usr/bin/dockerd --insecure-registry 0.0.0.0/0 
```

## 设置开机并重启docker

```bash
systemctl enable docker
systemctl daemon-reload
systemctl restart docker
docker login --username=admin --passwrod=seentao registry-host:port
```
## 安装docker-bash

```bash
yum install wget -y
wget -P ~ https://github.com/yeasy/docker_practice/raw/master/_local/.bashrc_docker
echo "[ -f ~/.bashrc_docker ] && . ~/.bashrc_docker" >> ~/.bashrc
source ~/.bashrc
```

---
