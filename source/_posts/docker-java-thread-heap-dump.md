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

```bash
docker exec -it CONTAINER_NAME bash
```
## 查看java进程

```bash
jps
```
## 转存进程Thread信息

```bash
jstack PID > thread.tdump
```
## 转存进程Heap信息

```bash
jmap -dump:live,format=b,file=heapDump.hprof PID
```
<!-- more -->
## 从容器中拷贝出文件

```bash
sudo docker cp CONTAINER_NAME:threadDump.tdump .
sudo docker cp CONTAINER_NAME:heapDump.hprof .
```
## 进行内存分析
使用JProfiler、[MAT](http://www.eclipse.org/mat/downloads.php)等工具进行内存分析

---