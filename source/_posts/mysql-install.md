---
title: centos7安装mysql
date: 2017-12-04 21:45:19
categories: Mysql
tags: 
- centos7
- mysql
---
## 安装mysql-rpm包
直接安装:

```bash
rpm -Uvh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
```
或者下载到本地后安装：

```bash
wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
yum localinstall mysql-community-release-el7-5.noarch.rpm
```
验证是否安装成功：

```bash
yum repolist enabled | grep "mysql.*-community.*"
```
<!-- more -->
## 安装mysql-server
 
```bash
yum install mysql-community-server  -y
rpm -qi mysql-community-server.x86_64 0:5.6.24-3.el7
```
## 相关设置
### 修改字符集

```bash
vi /etc/my.cnf
```
在`[mysqld]`下添加`character_set_server = utf8`，启动后`SHOW VARIABLES LIKE 'character%';`查看字符集信息
### 设置root密码
`systemctl start mysqld`启动服务后
`mysqladmin -u root password 'new-passwd'`
### 开启远程访问

```sql
grant all privileges on *.* to 'root'@'%' identified by 'passwd' with grant option;
FLUSH PRIVILEGES; 
```
###安全设置（可选）

```
mysql_secure_installation;
```

---