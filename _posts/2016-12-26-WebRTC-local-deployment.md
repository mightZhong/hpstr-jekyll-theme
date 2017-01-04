---
layout: post
title: WebRTC（AppRTC） 本地部署
description: "WebRTC local deployment guide."
tags: [webrtc]
comments: true
---

## 虚拟机配置


建议安装64位Ubuntu-14.04.1-desttop LTS版，apprtc官方安装说明上也提及了该版本，异常问题较少，整个过程需要翻墙，设置好代理

### 设置系统代理

/etc/profile末尾添加  

~~~  
 http_proxy= http://10.159.32.155:8080  
 https_proxy= http://10.159.32.155:8080  
 ftp_proxy= ftp://10.159.32.155:8080  
 export http_proxy https_proxy ftp_proxy  
~~~  

保存退出后执行 `$ source /etc/profile` 更新生效  

### 设置apt-get代理
/etc/apt/apt.conf(没有就自己创建)末尾添加   

~~~  
 Acquire:: http::Proxy "http://10.159.32.155:8080";  
 Acquire:: https::Proxy "https://10.159.32.155:8080 ";  
 Acquire:: ftp::Proxy "ftp://10.159.32.155:8080 ";  
~~~
 
保存退出后更新源 `$ apt-get -o Debug:: Acquire::http=true update`

### 设置 git 代理（临时）  
`$ git config --global http.proxy http://10.159.32.155:8080`  
`$ git config --global https.proxy http://10.159.32.155:8080`

### 设置npm代理（临时） 
`$ npm config  set   proxy http://10.159.32.155:8080`  
`$ npm config  set   https-proxy http://10.159.32.155:8080`  

---

## 安装apprtc  

### 下载源码 
`$ git clone https://github.com/webrtc/apprtc.git`  
不要下载到共享文件夹中，否则可能出现各种异常问题

### 安装方法  
参考README.md  
安装grunt之前先安装[nodejs](https://www.npmjs.com/)，npm是nodejs的管理工具  
`$ sudo apt-get install nodejs`  
`$ sudo npm install -g npm`  
如需升级node,使用nvm，具体参考[帖子](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-an-ubuntu-14-04-server)  

在Ubuntu14.04版本上默认安装是/usr/bin/nodejs，但是grunt需要的是/usr/bin/node，可以安装nodejs-legacy这个包解决。  
`$ sudo apt-get intsll nodejs-legacy`  
使用npm安装grunt-cli是最简单的方式,-g为全局安装  
`$ sudo npm install grunt-cli -g`  
最后进入apprtc目录，安装所有依赖包  
`$ sudo npm install`  

在这个阶段，可能会遇到一个问题  

> Downloading http://dl.google.com/closure-compiler/compiler-latest.tar.gz ...  Download failed: Error: connect ETIMEDOUT
 Unfortunately, ClosureCompiler.js could not be configured. See: https://github.com/dcodeIO/ClosureCompiler.js (create an issue maybe)

在[这个帖子](http://www.resolvinghere.com/sof/26271279.shtml)中找到一个不算很好的方案，但确实解决了问题。原因在于执行的脚本下载包时没有使用代理，方法是注释掉脚本中下载失败后退出语句，手动下载包并解压到指定路径，具体查看笔记*《closure-compiler install error》*
 
 在Ubuntu上，还需要安装webtest  
 `$ sudo apt-get install python-webtest`  
 在启动AppRTC或每次修改代码后，需重新编译  
 `$ sudo grunt build`  
 本地执行时提示找不到Java,需安装Java 运行环境  
`$ sudo apt-get install default-jre`  
 修改两个文件  
*apprtc/src/app_engine/constants.py*  

~~~  
 <== TURN_BASE_URL = 'https://computeengineondemand.appspot.com'  
 ==> TURN_BASE_URL = 'http://192.168.67.130:8083'  
 <== ICE_SERVER_BASE_URL = 'https://networktraversal.googleapis.com'  
 ==> ICE_SERVER_BASE_URL = 'http://192.168.67.130:8083'  
 <== ICE_SERVER_API_KEY = os.environ.get('ICE_SERVER_API_KEY')  
 ==> ICE_SERVER_API_KEY = '4080218913'  
 <== WSS_INSTANCE_HOST_KEY: 'apprtc-ws-2.webrtc.org:443'  
 ==> WSS_INSTANCE_HOST_KEY: '192.168.67.130:8089'  
~~~

*apprtc/scr/app_engine/apprtc.py*  

~~~
 get_wss_parameters(request)函数下的wss和https 改为 ws和http
~~~

修改完保存后重新编译一下 `$ sudo grunt build`

### 安装Google App Engine SDK for Python  
AppRTC的运行需要使用 Google App Engine，[安装包下载](https://cloud.google.com/appengine/downloads#Google_App_Engine_SDK_for_PHP)，安装说明详见下载页面  
 `$ unzip google_appengine_1.9.34.zip`  
 添加到环境变量  
 `$ export PATH=$PATH:/opt/webrtc/google_appengine/`  
 确认python是2.7版本  
 `$ /usr/bin/env python -v`  

### 启动AppRTC 
虚拟机内部使用静态地址，防止重启后IP改变  
 修改/etc/hosts配置文件，增加：  

~~~
 192.168.67.130  apprtc.wanttosee.com
~~~
	
后面所有IP都用域名表示，方便修改部署。  
运行Google App Engine SDK dev server启动AppRTC  
`$ <path to sdk>/dev_appserver.py --host apprtc.wanttosee.com --port 8085 ./out/app_engine`  
因为是为公网、或者局域网其他人提供服务，所以这里 host=0.0.0.0，以避免只监听 127.0.0.1 的状况, 端口默认为8080,修改为8085

在设置代理的情况下如果报以下错误：  

> HTTPError: HTTP Error 502: Proxy Error ( Connection refused )

此时需关闭代理，一般情况下新启一个终端，查看环境变量，如无代理即可  
除了这个问题，还有一个常见的warnning：  
 
> Invalid or no value returned from memcache, using fallback: null

---

## 安装collider(信令服务器)  
### 安装golang  
安装信令服务器（Collider，在apprtc的src目录下）依赖golang，而安装golang可以使用gvm(go version manger),参考[帖子](https://mozillazg.com/2014/12/go-use-gvm-to-manage-go-version.html)  

查看可安装golang版本  
`$ gvm listall`

安装golang（注意：在安装1.4版本的基础上才能安装1.5版本）  
`$ gvm  install  go1.4.2`  
`$ gvm  use  go1.4.2`  
`$ gvm  install  go1.5`     
`$ gvm  use  go1.5`   

安装路径：~/.gvm/gos/go1. 

设置环境变量GOROOT & GOPATH，确认这俩被正确设置
GOROOT:Go的安装路径  
`$ export GOROOT=~/.gvm/gos/go1.5`  
GOPATH:Go从1.1版本开始必须设置这个变量，而且不能和Go的安装目录一样，创建/opt/go，该目录下再创建三个文件夹src、bin、pkg  
`$ export GOPATH=/opt/go`

### 安装collidermain  
源码存在于/apprtc/src/collider，安装说明可查看README.md  
`$ ln -rs /opt/webrtc/apprtc/src/collider/collider $GOPATH/src/`  
`$ ln -rs /opt/webrtc/apprtc/src/collider/collidermain $GOPATH/src/`  
`$ ln -rs /opt/webrtc/apprtc/src/collider/collidertest $GOPATH/src/`  

创建软连接注意使用全路径  
`$ go get collidermain`  
提示：unrecognized import path "golang.org/x/net/websocket"  
解决方法：下载websocket源码（github）到本地，放在$GOPATH/srcgolang.org/x/net/websocket目录下  
`$ go install collidermain`

此时可以在$GOPATH/bin下看到collidermain的执行文件
### 运行collidermain  
`$ ./collidermain -port=8089 -tls=false -room-server='http://apprtc.wanttosee.com:8085`

---

## 安装coTurn（打洞服务器）
### 下载安装包
根据系统选择： turnserver-3.2.2.1-debian-wheezy-ubuntu-mint-x86-64bits.tar.gz.[地址](https://code.google.com/archive/p/rfc5766-turn-server/downloads?page=1)  

### 安装
cat INSTALL 查看安装说明
```
sudo dpkg -i rfc5766-turn-server_3.2.5.4-1_amd64.deb
```
报错：
> rfc5766-turn-server depends on libevent-core- 2.0 - 5 (>= 2.0 .10 -stable); however:
Package libevent-core- 2.0 - 5 is not installed.

   **解决方法**：使用gdebi-core来替代默认的dpkg， 它可以自动安装依赖库  
`$ sudo dpkg -r rfc5766-turn-server`  
`$ sudo apt-get install gdebi-core`  
`$ udo gdebi rfc5766-turn-server_3 .2 .5 .4 - 1 _amd64.deb` 

### 配置  
根据需求，修改配置文件/etc/turnserver.conf  

~~~  
\# 不多讲  
listening-device=eth0  
listening-port=3478  
relay-device=eth0  
\# 记得开防火墙  
min-port=59000  
max-port=65000  
\# 更繁杂的输出日志  
Verbose  
\# WebRTC 的消息里会用到  
fingerprint  
\# WebRTC 认证需要  
lt-cred-mech  
\# REST API 认证需要  
use-auth-secret  
\# \使用“静态”的 KEY，Google 自己也用的这个  
static-auth-secret=4080218913  
\# 用户登录域  
realm=http://apprtc.wanttotsee.com:8085  
\# 可为 TURN 服务提供更安全的访问  
stale-nonce  
\# SSL 需要用到的, 生成命令:  
\# sudo openssl req -x509 -newkey rsa:2048 -keyout /etc/turn_server_pkey.pem -out  #/etc/turn_server_cert.pem -days 99999 -nodes  
cert=/etc/turn_server_cert.pem  
pkey=/etc/turn_server_pkey.pem  
\# 屏蔽 loopback, multicast IP地址的 relay  
no-loopback-peers  
no-multicast-peers  
\# 启用 Mobility ICE 支持(具体干啥的我也不晓得)  
mobility  
\# 禁用本地 telnet cli 管理接口  
no-cli  
~~~

### 运行coTurn
修改/etc/default/rfc5766-turn-server,  
去掉TURNSERVER_ENABLE=1的注释  
`$ sudo service rfc5766-turn-server start`

---

## 测试调试  
本地浏览器（firefox或chrome）地址栏输入 http://192.168.67.170:8085 能建立视频会话，但是页面上会提示信息  

> No TURN server; unlikely that media will traverse networks. If this persists please [report it to discuss-webrtc@googlegroups.com]
 
这是由于没有完成TURN REST API服务器，apprtc需要获得TURN服务器的URL和证书才能与TURN建立连接，该服务器就是用户返回这些信息，采用RESTful标准。 

简单的TURN REST API服务器
 
~~~ js
var express = require('express');
var crypto = require('crypto');
var app = express();

var hmac = function(key, content){
    var method = crypto.createHmac('sha1', key);
    method.setEncoding('base64');
    method.write(content);
    method.end();
    return method.read();
};

//app.get('/v1alpha/iceconfig', function(req, resp) {
app.post('/v1alpha/iceconfig', function(req, resp) {
console.log('Get connection....');
var query = req.query;
var key = '4080218913'; // 这里的 key 是事先设置好的, 我们把他当成一个常亮来看, 所以就不从HTTP请求参数里读取了

//if (!query['username']) {
//    return resp.send({'error':'AppError', 'message':'Must provide username.'});
//} else {
    var time_to_live = 600;
    var timestamp = Math.floor(Date.now() / 1000) + time_to_live;
    var turn_username = timestamp + ':' + query['username'];
    var password = hmac(key, turn_username);
    //解决跨域问题
    resp.header('Access-Control-Allow-Origin', '*');
    resp.header('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE,OPTIONS');
    resp.header('Access-Control-Allow-Headers', 'Content-Type, Authorization, Content-Length, X-Requested-With');
    return resp.send({
        iceServers: 
        {
            urls:[
                "turn:apprtc.wanttotsee.com:3478?transport=udp",
                "turn:apprtc.wanttotsee.com:3478?transport=tcp",
                "turn:apprtc.wanttotsee.com:3479?transport=udp",
                "turn:apprtc.wanttotsee.com:3479?transport=tcp"
                ],
                username:turn_username,
                //password:password,
                credential:password,
                ttl:time_to_live
       }
    });
//}

});

app.listen('3033', function(){
    console.log('server started');
});
~~~

关于跨域策略可查看文章 [AJAX](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000/001434499861493e7c35be5e0864769a2c06afb4754acc6000)   
可以使用curl命令测试该服务器，看返回是否正常。  

`$ curl -X POST "http://apprtc.wanttosee.com:3033/v1alpha/iceconfig?key=4080218913"` 

提示：apprtc项目TURN相关部分在2016年左右做了重构，该为IceServers，具体可查看github上的提交记录。 
网上大部分教程都是2015年写的，所以使用最新代码运行会出现问题，需要做相应修改，例如：  

- TURN REST API请求方式由GET变为POST，所以服务器app.get 需改成 app.post，否则会报404 not found 错误  
- 返回的HTTP头需要加上'Access-Control-Allow-Origin',  '*'，否则有跨域问题  
- 返回的JSON对象格式也有了变化，可参考utils_test.js  

### Nginx  
用来统一 HTTP 对外服务端口，以及最主要的反代 3033 TURN REST API服务 
 
- 先装好nginx：apt-get install nginx -y  
- 默认配置文件在 /etc/nginx/sites-enabled/default 。打开后，新增：  

~~~
server {  
      listen 8083;  
      server_name 192.168.67.130;
      access_log /var/log/nginx/server.access.log combined;
      location / {
        proxy_pass http://192.168.67.130:8085;
        proxy_set_header Host $host;
      }
      location /v1alpha/iceconfig {
        proxy_pass http://192.168.67.130:3033;
        proxy_set_header Host $host;
      }
}
~~~  

- 使用 service nginx reload 重载配置文件
- 日志保存在/var/log/nginx路径下

---

## 总结
- 房间服务器(端口8085)运行：  
`$ dev_appserver.py --host apprtc.wanttosee.com --port 8085 ./out/app_engine`

- 信令服务器(端口8089)运行：  
`$ ./collidermain -port=8089 -tls=false -room-server='http://apprtc.wanttosee.com:8085'`  
- 打洞服务器(端口3478)运行：
`$ sudo service rfc5766-turn-server start`  

  ``` seq
dev_appserver(8085)->nginx(8083):reset api
nginx(8083)->turn_reset_api(3033):reset api
dev_appserver(8085)->collidermain(8089):
dev_appserver(8085)->turnserver(3478):
dev_appserver(8085)->nginx(8083):
```  

---

## 参考  
- http://www.jianshu.com/p/c55ecf5a3fcf  












 