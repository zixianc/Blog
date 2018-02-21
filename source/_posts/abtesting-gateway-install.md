---
title: 动态策略灰度发布系统（ABTestingGateway部署和使用）
date: 2018-02-11 12:11:44
categories:
- Nginx
tags: 
- nginx
- 动态灰度发布
---
## [部署ABTestingGateway](https://github.com/CNSRE/ABTestingGateway)

```bash
git clone https://github.com/CNSRE/ABTestingGateway.git
cd /path/to/ABTestingGateway/utils && mkdir logs
```
## 安装openresty
安装依赖

```bash
wget https://openresty.org/download/openresty-1.9.7.5.tar.gz
yum install pcre-devel -y
yum install openssl-devel -y
```
编译

```bash
./configure
gmake && gmake install
```
编译完成配置nginx的环境变量
<!-- more -->
## 安装redis

```bash
wget http://download.redis.io/releases/redis-4.0.6.tar.gz
tar -zxvf redis-4.0.6.tar.gz
cd /path/to/redis-4.0.6
./make MALLOC=libc
```
配置src环境变量

```bash
cd /path/to/ABTestingGateway/utils
# 该redis.conf文件默认为protected-mode模式,关闭才可远程连接
redis-server conf/redis.conf
```
## 部署测试
按照github上的例子进行测试

```bash
# 启动upstream server，其中stable为默认upstream
1. /usr/local/nginx/sbin/nginx -p `pwd` -c conf/stable.conf
2. /usr/local/nginx/sbin/nginx -p `pwd` -c conf/beta1.conf
3. /usr/local/nginx/sbin/nginx -p `pwd` -c conf/beta2.conf
4. /usr/local/nginx/sbin/nginx -p `pwd` -c conf/beta3.conf
5. /usr/local/nginx/sbin/nginx -p `pwd` -c conf/beta4.conf

# 启动灰度系统，proxy server，灰度系统的配置也写在conf/nginx.conf中
6. /usr/local/nginx/sbin/nginx -p `pwd` -c conf/nginx.conf

# 简单验证：添加分流策略组
$ curl 127.0.0.1:8080/ab_admin?action=policygroup_set -d '{"1":{"divtype":"uidsuffix","divdata":[{"suffix":"1","upstream":"beta1"},{"suffix":"3","upstream":"beta2"},{"suffix":"5","upstream":"beta1"},{"suffix":"0","upstream":"beta3"}]},"2":{"divtype":"arg_city","divdata":[{"city":"BJ","upstream":"beta1"},{"city":"SH","upstream":"beta2"},{"city":"XA","upstream":"beta1"},{"city":"HZ","upstream":"beta3"}]},"3":{"divtype":"iprange","divdata":[{"range":{"start":1111,"end":2222},"upstream":"beta1"},{"range":{"start":3333,"end":4444},"upstream":"beta2"},{"range":{"start":7777,"end":2130706433},"upstream":"beta2"}]}}'

{"desc":"success ","code":200,"data":{"groupid":0,"group":[0,1,2]}}

# 简单验证：设置运行时策略

$ curl "127.0.0.1:8080/ab_admin?action=runtime_set&hostname=api.weibo.cn&policygroupid=0"

# 分流
$ curl 127.0.0.1:8030 -H 'X-Uid:39' -H 'X-Real-IP:192.168.1.1'
this is stable server

$ curl 127.0.0.1:8030 -H 'X-Uid:30' -H 'X-Real-IP:192.168.1.1'
this is beta3 server

$ curl 127.0.0.1:8030/?city=BJ -H 'X-Uid:39' -H 'X-Real-IP:192.168.1.1'
this is beta1 server
```

## 配置文件说明
```bash
cd /path/to/ABTestingGateway/utils/conf
```
关键配置文件为nginx.conf、default.conf、upstream.conf、vhost.conf

* nginx.conf为nginx入口配置文件,在这里引入包含其他配置文件
* default.conf为管理后台配置文件
* upstream.conf为upstream配置文件
* vhost.conf为代理转发配置文件模板

## ABTestingGateway多环境多项目配置
* 复制一份vhost.conf备用,然后修改vhost.conf,修改后如下(去掉server部分)

```bash
lua_shared_dict api_root_sysConfig 1m;
lua_shared_dict kv_api_root_upstream 100m;

lua_shared_dict api_abc_sysConfig 1m;
lua_shared_dict kv_api_abc_upstream 100m;
```
* 配置upstream，假设有两个环境三个项目

```bash
#####gray env#####

upstream  beta1_login  {
    server    10.10.16.147:3000    weight=1 fail_timeout=10 max_fails=1;
    keepalive    1000;
}

upstream  beta1_auth  {
    server    10.10.16.147:3002    weight=1 fail_timeout=10 max_fails=1;
    keepalive    1000;
}

upstream beta1_api {
    server    10.10.16.140:8080    weight=1 fail_timeout=10 max_fails=1;
    keepalive    1000;
}

#####prod env#####

upstream  stable_login  {
    server    10.10.16.106:3000    weight=1 fail_timeout=10 max_fails=1;
}

upstream  stable_auth  {
    server    10.10.16.106:3002    weight=1 fail_timeout=10 max_fails=1;
}

upstream stable_api {
    server    10.10.16.111:8080    weight=1 fail_timeout=10 max_fails=1;
    keepalive    1000;
}
```
* 配置转发规则，删除之前vhost.conf的备用文件中在第一步骤的内容，分别复制命令为login.conf、auth.conf、api.conf。修改listen、server_name；stable替换为upstream中配置的名称

```bash
server {
        listen       80;
        server_name  tlogin.abtesting.com;

        access_log logs/vhost_access.log  main;
        error_log  logs/vhost_error.log;

        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;

        set $redis_host '127.0.0.1';
        set $redis_port '6379';
        set $redis_uds '/var/run/redis.sock';
        set $redis_connect_timeout 10000;
        set $redis_dbid 0;
        set $redis_pool_size 1000;
        set $redis_keepalive_timeout 90000;

        location ~*  /abc/(i|f)/ {
                set $hostkey $server_name.abc;
                set $sysConfig api_abc_sysConfig;
                set $kv_upstream kv_api_abc_upstream;
                set $backend 'stable_login';
                rewrite_by_lua_file '../diversion/diversion.lua';
                proxy_pass http://$backend;
        }

        location / {

                error_log  logs/vhost_error.log debug;

                set $hostkey $server_name;
                set $sysConfig api_root_sysConfig;
                set $kv_upstream kv_api_root_upstream;
                set $backend 'stable_login';
                rewrite_by_lua_file '../diversion/diversion.lua';
                rewrite ^(.*)$  https://$host$1 permanent;
                proxy_pass http://$backend;
        }
}
```
* 添加ab规则，以city_arg规则为例
为保证测试的一直性，将redis的数据清空，进入redis-cli执行`flushdb`，关闭所有nginx进行执行`kill -9 pgrep nginx`，操作完后添加规则并设置hostname对应的规则

```bash
curl 127.0.0.1:8080/ab_admin?action=policy_set -d '{"divtype":"arg_city","divdata":[{"city":"BJ","upstream":"beta1_api"}]}'
curl "127.0.0.1:8080/ab_admin?action=runtime_set&hostname=tapi.abtesting.com&policid=0"
curl 127.0.0.1:8080/ab_admin?action=policy_set -d '{"divtype":"arg_city","divdata":[{"city":"BJ","upstream":"beta1_login"}]}'
curl "127.0.0.1:8080/ab_admin?action=runtime_set&hostname=tlogin.abtesting.com&policyid=1"
curl 127.0.0.1:8080/ab_admin?action=policy_set -d '{"divtype":"arg_city","divdata":[{"city":"BJ","upstream":"beta1_auth"}]}'
curl "127.0.0.1:8080/ab_admin?action=runtime_set&hostname=tauth.abtesting.com&policyid=2"
```
* 启动服务，测试规则

```bash
cd /path/to/ABTestingGateway/utils
nginx -p `pwd` -c conf/nginx.conf
```
在浏览器中输入tapi.abtesting.com将转发到*stable_api*配置的upstream，输入`tapi.abtesting.com?city=BJ`将转发到*beta1_api*配置的upstream，`tlogin.abtesting.com`和`tauth.abtesting.com`同理，这里域名可以修改主机的hosts文件来绑定
## https支持
以`https://tapi.abtesting.com`为例，修改api.conf
将80端口的server复制一份修改监听端口为443，在server_name下面添加如下配置

```bash
ssl                  on;
ssl_certificate      abtesting.com_bundle.crt;
ssl_certificate_key  abtesting.com.key;
ssl_session_timeout  5m;
ssl_protocols  SSLv3 TLSv1;
ssl_ciphers  HIGH:!ADH:!EXPORT56:RC4+RSA:+MEDIUM;
ssl_prefer_server_ciphers   on;
```
其中`abtesting.com_bundle.crt`和`abtesting.com.key`分别为私钥和公钥的文件名，放置在nginx.conf的同目录中，最终为

```bash
server {
        listen       443;
        server_name  tlogin.abtesting.com;
        ssl                  on;
        ssl_certificate      abtesting.com_bundle.crt;
        ssl_certificate_key  abtesting.com.key;
        ssl_session_timeout  5m;
        ssl_protocols  SSLv3 TLSv1;
        ssl_ciphers  HIGH:!ADH:!EXPORT56:RC4+RSA:+MEDIUM;
        ssl_prefer_server_ciphers   on;


        access_log logs/vhost_access.log  main;
        error_log  logs/vhost_error.log;

        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;

        set $redis_host '127.0.0.1';
        set $redis_port '6379';
        set $redis_uds '/var/run/redis.sock';
        set $redis_connect_timeout 10000;
        set $redis_dbid 0;
        set $redis_pool_size 1000;
        set $redis_keepalive_timeout 90000;

        location ~*  /abc/(i|f)/ {
                set $hostkey $server_name.abc;
                set $sysConfig api_abc_sysConfig;
                set $kv_upstream kv_api_abc_upstream;
                set $backend 'stable_login';
                rewrite_by_lua_file '../diversion/diversion.lua';
                proxy_pass http://$backend;
        }

        location / {

                error_log  logs/vhost_error.log debug;

                set $hostkey $server_name;
                set $sysConfig api_root_sysConfig;
                set $kv_upstream kv_api_root_upstream;
                set $backend 'stable_login';
                rewrite_by_lua_file '../diversion/diversion.lua';
                proxy_pass http://$backend;
        }
}
```
修改完成后重启nginx，执行

```bash
nginx -p `pwd` -c conf/nginx.conf -s reload
```
在浏览器中输入`https://tapi.abtesting.com`和`https://stapi.abtesting.com?city=BJ`进行测试，如需将http重定向为https，在80监听的`location /`中添加`rewrite ^(.*)$  https://$host$1 permanent;`
## websocket支持 
* 在upstream中添加websocket的upstream

```bash
upstream  beta1_websocket  {
    server    10.10.16.91:8899    weight=1 fail_timeout=10 max_fails=1;
}
upstream  stable_websocket  {
    server    10.10.16.117:8899    weight=1 fail_timeout=10 max_fails=1;
}
```
* 复制一份api.conf命名为websocket.conf，修改server_name和stable对应的配置，分别在80和443监听中`location /`位置添加如下配置

```bash
proxy_http_version 1.1;
proxy_read_timeout 7200s;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```
* 在nginx.conf文件中添加如下配置，重启nginx

```bash
map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
}
```
* 添加websocket的AB规则并绑定

```bash
curl 127.0.0.1:8080/ab_admin?action=policy_set -d '{"divtype":"arg_city","divdata":[{"city":"BJ","upstream":"beta1_websocket"}]}'
curl "127.0.0.1:8080/ab_admin?action=runtime_set&hostname=twebsocket.abtesting.com&policid=3"
```
* 测试websocket，如下为websocket的h5客户端，修改ws的地址后打开页面进行测试

```html
<!DOCTYPE HTML>
<html>
<head>
    <title>My WebSocket</title>
</head>

<body>
Welcome<br/>
<input id="text" type="text" /><button onclick="send()">Send</button>    <button onclick="closeWebSocket()">Close</button>
<div id="message">
</div>
</body>

<script type="text/javascript">
    var websocket = null;

    //判断当前浏览器是否支持WebSocket
    if('WebSocket' in window){
        websocket = new WebSocket("wss://twebsocket.abtesting.com/websocket?city=BJ");
    }
    else{
        alert('Not support websocket')
    }

    //连接发生错误的回调方法
    websocket.onerror = function(){
        setMessageInnerHTML("error");
    };

    //连接成功建立的回调方法
    websocket.onopen = function(event){
        setMessageInnerHTML("open");
        setMessageInnerHTML(getNowFormatDate());
    }

    //接收到消息的回调方法
    websocket.onmessage = function(event){
        setMessageInnerHTML(event.data);
    }

    //连接关闭的回调方法
    websocket.onclose = function(){
        setMessageInnerHTML("close");
        setMessageInnerHTML(getNowFormatDate());
    }

    //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function(){
        websocket.close();
    }

    //将消息显示在网页上
    function setMessageInnerHTML(innerHTML){
        document.getElementById('message').innerHTML += innerHTML + '<br/>';
    }

    //关闭连接
    function closeWebSocket(){
        websocket.close();
    }

    // 获取当前日期
    function getNowFormatDate() {
        var date = new Date();
        var seperator1 = "-";
        var seperator2 = ":";
        var month = date.getMonth() + 1;
        var strDate = date.getDate();
        if (month >= 1 && month <= 9) {
            month = "0" + month;
        }
        if (strDate >= 0 && strDate <= 9) {
            strDate = "0" + strDate;
        }
        var currentdate = date.getFullYear() + seperator1 + month + seperator1 + strDate
                + " " + date.getHours() + seperator2 + date.getMinutes()
                + seperator2 + date.getSeconds();
        return currentdate;
    }

    //发送消息
    function send(){
        var message = document.getElementById('text').value;
        websocket.send(message);
    }
</script>
</html>
```
地址为`wss://twebsocket.abtesting.com/websokcet`将转发到*stable_websocket*，`wss://twebsocket.abtesting.com/websokcet?city=BJ`将转发到*beta1_websocket*

---