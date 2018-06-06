---
layout: page
title:  "一个诡异的 404 请求"
date:   2018-06-06 16:30:00 +0800
category: tech nginx
permalink: stracenginx.html
---

### 发现问题
由于其他项目组使用了我们的视频服务，期间出现问题经常来找我（～）。最近告诉我说有个文件访问时不时报 404，他们各种追踪查不到问题。

### 问题重现 && 定位问题
* 问题重现  
重现这个问题还比较简单，直接 `curl` 命令请求那个 url，确实有时 200 ，有时 404 
* 定位问题  
从 `curl —I` 返回的header中看不到什么有用的信息。由于问题的偶发性，开始我以为是 nginx 的负载均衡机器中某台机器访问有问题。但是看了下线上只有两台机器，分别用 `curl -x` 绑定测试也都出现了 404
然后去线上看 nginx 错误日志，把日志级别改成 debug，观察到 nginx 返回 200 和 404 的请求如下：  
```bash
2018/06/05 16:37:33 [notice] 14434#0: *134482 "/(.*)$" matches "/5b05286d83ba88cd5c875907/H.mp4" while sending to client, client: 10.100.136.218, server: video.xxxx.com, request: "GET /5b05286d83ba88cd5c875907/H.mp4 HTTP/1.0", host: "video.xxxx.com", referrer: "http://www.xxxx.com/topic/micro?sid=iw3di1"
2018/06/05 16:37:33 [notice] 14434#0: *134482 rewritten data: "/K12/M00/04/E2/CmSEcVsFL-GAaOm1Agcnmw2-Qfs186.mp4", args: "" while sending to client, client: 10.100.136.218, server: video.xxxx.com, request: "GET /5b05286d83ba88cd5c875907/H.mp4 HTTP/1.0", host: "video.xxxx.com", referrer: "http://www.xxxx.com/topic/micro?sid=iw3di1"
2018/06/05 16:39:47 [info] 14432#0: *134485 client 10.98.16.109 closed keepalive connection
```
```bash
2018/06/05 16:37:26 [info] 14434#0: *134481 client 10.98.16.109 closed keepalive connection
```

可以看到下面的 404 请求只有一条 `closed keepalive connection`，难道请求还没有被 nginx 接收处理？  
或者说， nginx 处理请求的时候发生了错误，但是并没有记录日志！  
事实证明，确实是后者。
### 问题根源
由于 nginx 日志能看到的内容有限，要知道 nginx 有没有处理请求，只能另想其他办法了。超哥指导下，用 `strace` 跟踪一下 nginx 的 worker 进程。果然发现了问题  
下面是用 `strace` 跟踪的 200 和 404 请求  
 请求 200
![200](/assets/post-images/strace/200.png)
 请求 404
![404](/assets/post-images/strace/404.png)
上面的图，就可以看出来问题所在了， nginx 内部请求 redis 的时候有时返回成功，有时返回空！  
去 redis 验证一下，果然  
![404](/assets/post-images/strace/redis.png)
### 甩锅指南（😄）
所以一开始的判断还是没错的，确实是负载均衡某台机器出现了问题  
虽然不是 nginx 是 redis 
然后老大成功把这个 redis 的锅领走～