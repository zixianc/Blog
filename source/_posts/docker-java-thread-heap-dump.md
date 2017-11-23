---
title: docker-java-thread&heap-dump
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
