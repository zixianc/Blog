---
title: 获取docker容器内java进程的堆栈信息
date: 2017-11-23 09:57:36
categories:
- Docker
tags:
- docker
- java
---
## 进入docker容器内
```
docker exec -it CONTAINER_NAME bash
```
## 查看java进程
```
jps
```
## 转存进程Thread信息
```
jstack PID > thread.tdump
```
## 转存进程Heap信息
```
jmap -dump:live,format=b,file=heapDump.hprof PID
```
<!-- more -->
## 从容器中拷贝出文件
```
sudo docker cp CONTAINER_NAME:threadDump.tdump .
sudo docker cp CONTAINER_NAME:heapDump.hprof .
```
## 使用MAT进行内存分析
[MAT下载地址](http://www.eclipse.org/mat/downloads.php)