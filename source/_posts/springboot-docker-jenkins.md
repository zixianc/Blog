---
title: jenkins+docker持续集成构建SpringBoot项目
date: 2017-12-06 22:48:22
categories: SpringBoot
tags: 
- springboot
- docker
- jenkins
---
## SpringBoot项目配置
假设要构建一个用户中心服务，新建一个名为user-center的SpringBoot项目
### 在springboot项目pom.xml中添加依赖
```xml
<!-- docker打包 -->
<plugin>
	<groupId>com.spotify</groupId>
	<artifactId>docker-maven-plugin</artifactId>
	<version>0.4.13</version>
	<configuration>
		<serverId>docker-hosted</serverId>
		<!-- docker仓库地址，用于推送镜像 -->
		<registryUrl>${docker.repository}</registryUrl>
		<pushImage>true</pushImage>
		<!-- Dockerfile路径 -->
		<dockerDirectory>src/main/docker</dockerDirectory>
		<!-- 构建的镜像名称 -->
		<imageName>${docker.repository}/${project.artifactId}</imageName>
		<imageTags>
			<imageTag>latest</imageTag>
		</imageTags>
		<resources>
			<resource>
				<targetPath>/</targetPath>
				<directory>${project.build.directory}</directory>
				<include>${project.build.finalName}.jar</include>
			</resource>
		</resources>
	</configuration>
</plugin>
```
<!-- more -->
docker仓库地址可以直接写在pom文件中，但为了多环境配置可以将地址配置在maven-settings.xml

```xml
<profiles>
  <profile>    
    <id>nexus</id>    
  <properties>
   <docker.repository>10.10.16.160:8083</docker.repository>
 </properties>
</profile>
</profiles>

<activeProfiles>    
 <activeProfile>nexus</activeProfile>    
</activeProfiles>
```
### 创建Dockerfile
```bash
FROM java:8 # 基础镜像（主程序镜像）
VOLUME /tmp # 挂在tmp盘
ADD user-center-0.0.1-SNAPSHOT.jar workdir/app.jar
WORKDIR workdir # 指定工作目录
ENV JAVA_OPTS=""
ENTRYPOINT [ "sh", "-c", "java -Djava.security.egd=file:/dev/./urandom -Duser.timezone=GMT+08 $JAVA_OPTS -jar /workdir/app.jar" ] # 程序入口，支持动态传参
```
### 测试手动构建镜像
在命令行进入到项目路径执行`mvn clean package docker:build`来构建镜像，执行`mvn clean package docker:build -DpushImage`构建并推送镜像docker仓库
## Jenkins项目构建
新建一个名为user-center的maven项目
### 配置git仓库
![git仓库](http://p0kzweqn4.bkt.clouddn.com/jenkins1.png)
### 配置构建服务器
![添加构建服务器](http://p0kzweqn4.bkt.clouddn.com/jenkins2.png)
### 配置构建步骤
![构建步骤](http://p0kzweqn4.bkt.clouddn.com/jenkins3.png)
## 运行脚本
jenkins配置中用到的run-server脚本

```bash
#!/bin/bash
#私有仓库地址
REPO_URL=${1}
#项目名称
PROJ_NAME=${2}
#项目运行环境，多环境配置
PROFILE_ACTIVE=${3}
#eureka注册中心地址，未接入忽略该配置
EUREKA_URL=${4}

#停止原来的熔器
docker stop ${PROJ_NAME}
#删除原来的容器
docker rm ${PROJ_NAME}
#删除原来的镜像
docker rmi ${REPO_URL}/${PROJ_NAME}
#登录docker仓库
docker login --username=username --password=passwd ${REPO_URL}
#重新拉取新镜像
docker pull ${REPO_URL}/${PROJ_NAME}
#运行新的容器
docker run -itd \
--restart=always \
--net=host \
-m 2048m \
--name ${PROJ_NAME} \
-v /etc/localtime:/etc/localtime:ro \
--env JAVA_OPTS="-Dspring.profiles.active=${PROFILE_ACTIVE} -Deureka.client.serviceUrl.defaultZone=${EUREKA_URL} -server -Xms1024m -Xmx2048m  -XX:PermSize=64M -XX:MaxNewSize=256m -XX:MaxPermSize=128m -Djava.awt.headless=true -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp" \
${REPO_URL}/${PROJ_NAME}
```

---