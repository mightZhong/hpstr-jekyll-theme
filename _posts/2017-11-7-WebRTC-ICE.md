---
layout: post
title: WebRTC ICE详解
description: "WebRTC ICE详解."
tags: [webrtc]
comments: true
---

### ICE的作用
ICE(Interactive Connectivity Establishment)协议可使位于NAT后的客户端建立通信，另外ICE还提供通信许可验证功能，如果不完成ICE交换，就不会发送媒体，可防止浏览器被诱骗，将数据发送到非通信方的Internet主机，造成DOS泛洪攻击。   

### ICE呼叫流   
#### 1. 收集候选传输地址   
收集地址的过程必须在呼叫时进行，不能提前收集，当用户A向对端发起连接请求时，ICE代理A便开始收集候选地址，而当对端B收到连接请求后也将开始收集候选地址    
候选地址有三种：主机地址（从网卡上获取的本地地址），反射地址（从stun服务器中返回的最外侧NAT映射地址）， 中继地址（从TURN服务器中分配的地址），地址收集主要依靠stun技术，关于stun下面会介绍  

#### 2. 在信令通道中交换选项（Offer/Answer）
当收集到候选地址后，会对它们进行排序确定优先级，一般主机地址优先级最高，其次是反射地址，最后是中继地址。候选项与SDP中的一个特定媒体流相关联。Webrtc的默认行为是通过一个传输地址对所有媒体流进行传输，包括语言，视频和数据。在接收到SDP后，会将收到的地址与本地收集的地址，组成地址对并排序，用于后面的连接检查

#### 3. 执行连接检查
两端连接性测试,结果是一个4次握手过程

假设先从代理A向对端发送SDP,当对端代理B收到SDP时则开始执行连接检查，同时发送SDP响应给代理A，代理A接收到响应后，也开始执行连接检查。注意，ICE测试连通性用的端口就是RTC媒体会话的端口，这样当NAT打通成功后，就能相互发媒体了。   

候选项对最初处于“冻结状态”，一旦确定开始检查，则变为“等待”，然后根据优先级，最高优先级的地址通过stun连接检查发送到对端，状态变为“进行中”，如果被响应，则切换为“成功”，如果超时则为“失败”，继续检查下一地址。   

还有一种ICE优化技术“缓慢性ICE”,就是以尽可能小的候选项集合，先把ICE启动起来，并在此过程中，不断增加候选项，此功能目前仅在chrome上被支持。   

所有的ICE实现都要求与STUN(RFC5389)兼容,并且废弃Classic STUN(RFC3489).ICE的完整实现既生成checks(作为STUN client)    




#### 4. 选择选定的对并启动媒体
连接检查将一直持续，直到所有检查都已完成或某一对候选项被选中。选择对的操作由施控ICE代理执行。ICE协议有算法用于选择哪个浏览器是施控ICE代理，一般这样选择，如果有一方是ICE Lite 代理，那么这一端就是受控方，如果两端都是ICE full,那么发起Offer的一端就是施控方。   

至于如何选择地址对呢，根据地址对的优先级做检查，并将收到回复的地址对放入“有效地址对”中，然后根据local optimization策略选择一个地址对，然后将在USE-CANDIDATE标记，再次发送到对端。  

当受控ICE代理接收到施控ICE代理发来的stun连接检查的属性指示使用某个候选项时，会向对端回复响应，确认使用该对，此后两个浏览器就可以使用该确定的候选项对发送媒体

#### 5. 发送长连接（keepalive）   
  
为确保NAT映射和过滤规则不在媒体会话期间超时，ICE会通过使用中的候选项对发送连接检查，每隔15s发送一次。如果没有收到stun响应，则会停止媒体，重新启动ICE
#### 6. 重新启动ICE
当任何一端检测到使用中的地址发生变化，都重新启动ICE。如果用户重新加载浏览器页面，也会出现这一情况。

### STUN
STUN(Session Traversal Utilities for NAT)NAT会话穿透工具用于帮助进行NAT穿透。STUN可基于UDP,TCP,TLS传输，默认的UDP端口号3478.在核心STUN协议中，只有一种称为“Binding”的方法。Binding请求将导致NAT创建映射并分配公共IP和端口，也就是在NAT上打洞。在Binding响应中，由STUN识别的IP地址和端口号将通过XOR-MAPPED-ADDRESS属性返回。该属性会对ＩＰ地址和端口号执行异或运算，以实现模糊处理，防止应用层网关（ＡＬＧ）识别并重写这些信息。

### TURN
TURN服务器和TRUN客户端之间（通过NAT）的通信始终为UDP通信，默认端口3478.TURN客户端通过Allocate request向服务器请求中继地址，如果认证成功，服务器响应Allocate Success Responsse，并在XOR-RELAYED-ADDRESS字段中返回中继地址。该地址会通过SDP发送到通信对端。

### ICE Lite 
根据RFC, 如果ICE代理处于公网，那么它可以使用ICE Lite，也就是轻量级的ICE。ICE Lite不需要收集候选项，所有连接都使用主机地址，它也没有连通性测试，但是需要相应对端发来的连接测试请求。

### 如何确定controlling和controled
controlling的判断首先基于ice的设置，看看服务器是处于lite模式或者是full模式，lite模式时服务端总是controlled，而在full模式时，则根据收到offer或者是发起offer来判断，先发起offer的是controlling。

### 两种提名(regular nomination & aggressive nomination)

分为常规的提名和激进的提名。

 常规的提名在协商中controlling一端在发送binding request时候不会携带确认标志，当所有协商完成后对所有的candidate进行评估，最后会再次发送一个带有标志位的请求来表示确认。
而激进提名方式controlling在发送binding request的时候就会携带对应的标志位，当该次连通性测试完成时，就选定该连接，这种时候不会发送第二次的binding request    
 标志位

ICE扩展了以下几个stun的attribute，其中6.1中的标志位是USE_CANDIDATE，controlling在选择candidate的时候会在binding request中携带该标志位，而controlled在binding response和binding request都不会携带该标志位

### 利用STUN获取映射地址和进行连通性检查，分别发什么包
发的都是binding request,区别在于发给STUN服务器时不带身份验证，而发给远端client做连通性检查时需要短期身份验证

### ICE在SDP中新增的字段

	ICE Candidates   
	a=candidate:1467250027 1 udp 2122260223 192.168.0.196 46243 typ host generation 0   
	
	ICE Parameters   
	a=ice-ufrag:Oyef7uvBlwafI3hT   
	a=ice-pwd:T0teqPLNQQOf+5W+ls+P2p16   


### Note
由于webrtc强制使用ICE，所以原来SDP中的c(connection)中的IP和port将不被使用

#### 参考
《Webrtc权威指南》



 