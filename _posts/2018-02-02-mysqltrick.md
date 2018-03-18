---
layout: page
title:  "MySQL Trick"
date:   2018-02-02 17:40:02 +0800
category: tech database mysql
permalink: mysqltrick.html
---

> 设置sql结果输出分页 (more)

```
pager more
```

> 设置配置参数

```
# 全局参数
set global name=value;
set @@global.name:=value;

# 会话参数
set [session] name=value;
set @@session.name:=value;
```