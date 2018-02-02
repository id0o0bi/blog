---
layout: page
title:  "linux网络代理抓包工具折腾记 - mitmproxy"
date:   2018-01-03 16:16:01 +0800
categories: tech api
permalink: /blog/:year:month:day-meetmitmproxy.html
---

```
前段时间一直在给移动端写接口，一直没有找到linux下面趁手的代理抓包工具
毕竟windows下面有 Fiddler，mac下面有Charles。虽然界面有点臃肿，毕竟都还比较成熟。
```

```
蓝儿。他们都没有linux版本～
然后mitmproxy出现了，哈哈，要啥自行车～Linux真的什么都有
```

[mitmproxy官网](https://mitmproxy.org/)

参照安装说明安装完成后，直接在命令行启动 `mitmproxy`
启动后默认监听本机的 8080 端口，使用手机连接代理，即可看到手机上的网络请求
![Screenshot-from-2018-01-03-15-49-49](/assets/post-images/mitmproxy/Screenshot from 2018-01-03 15-49-49.png)
可以使用上下键选择请求，Enter键进入查看详情（也可以用Vim的hjkl快捷方向键）
详情分三个tab页展示，可以使用tab键切换，同样用上下键或hj浏览
* 第一个tab是请求的详情。可以看到这里有请求的header和body （这里body是空的）

```
assets/post-images/the hours/Virginia&Leonerd.jpg
```
![Screenshot-from-2018-01-03-15-50-36](/assets/post-images/mitmproxy/Screenshot from 2018-01-03 15-50-36.png)
* 第二个tab是响应的详情，里面有返回的header和body

![Screenshot-from-2018-01-03-15-50-49](/assets/post-images/mitmproxy/Screenshot from 2018-01-03 15-50-49.png)
* 第三个tab里有详细的请求过程(包括https的鉴权细节)

![Screenshot-from-2018-01-03-15-51-22](/assets/post-images/mitmproxy/Screenshot from 2018-01-03 15-51-22.png)

这些基本就满足了大部分接口测试的需求，另外它还支持修改参数重发等等。可以说很全面了