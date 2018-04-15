---
layout: page
title:  "mysql auto commit的坑"
date:   2018-04-15 16:42:01 +0800
category: tech api
permalink: /blog/:year:month:mysqlautocommit.html
---

#### 记录一下上周遇到的巨坑！！！

酱紫的：  
由于移动端的支付需求变更，需要对之前的业务流程做一些修改  
之前订单流程是下单完成后，返回一个支付宝或微信的支付订单参数给客户端，客户端直接调起微信支付宝支付，<b>也就是下单完成后没有其他数据库操作</b>（重点）  

这次修改后，下单之后仍需要对数据库进行一些更新操作  
于是按照正常逻辑写了代码在下单完成后，执行了两个update语句。update的sql在数据库执行测试没问题

我以为结束了。然后诡异的事情发生了，两个语句都没有执行了，数据库虽然返回成功，而且返回了受影响行数，但是数据却没有变化！！！

好，开始分析问题：  
一开始以为是mysql代理问题。于是把测试机器的代理去掉，直接连mysql数据库。并没有用，一切如旧。

好，然后上mysql数据库看错误日志。不得了，一直在不停报错
```
180412 10:46:55 [ERROR] Index idx_uid_uptime of learn_history/tbl_user_member_history_25 has 3 columns unique inside InnoDB, but MySQL is asking statistics for 4 columns. Have you mixed up .frm files from
 different installations? See http://dev.mysql.com/doc/refman/5.1/en/innodb-troubleshooting.html

180412 10:46:55 [ERROR] Index idx_uid_uptime of learn_history/tbl_user_member_history_26 has 3 columns unique inside InnoDB, but MySQL is asking statistics for 4 columns. Have you mixed up .frm files from
 different installations? See http://dev.mysql.com/doc/refman/5.1/en/innodb-troubleshooting.html

...
```
错误日志里面一大堆这样的日志，之前分表的索引有问题？但是跟这个在不同库，应该没影响啊。  
看来也不是这个问题。  

好，mysql确实是返回了affect_rows=1了，说明更新语句应该执行了，binlog里面肯定有吧。我来查。   
立刻把数据库上的binlog拉出来，查当时的日志记录：
```
> mysqldump mysql-bin.000002 -v > ~/bin.log
> less ~/bin.log

### 关键内容
#180412 10:24:35 server id 1  end_log_pos 53602792      Query   thread_id=9914  exec_time=0     error_code=0
use shizhan_trade/*!*/;
SET TIMESTAMP=1523499875/*!*/;
insert into tbl_***_trade( **** ) values ( **** )

#180412 10:24:35 server id 1  end_log_pos 53603389      Query   thread_id=9914  exec_time=0     error_code=0
SET TIMESTAMP=1523499875/*!*/;
insert into tbl_***_goods( **** ) values ( **** )

#180412 10:24:35 server id 1  end_log_pos 53603484      Query   thread_id=9815  exec_time=0     error_code=0
SET TIMESTAMP=1523499875/*!*/;
SET @@session.sql_mode=2097152/*!*/;
BEGIN

#180412 10:24:35 server id 1  end_log_pos 53603627      Query   thread_id=9815  exec_time=0     error_code=0
use jira/*!*/;
SET TIMESTAMP=1523499875/*!*/;
DELETE FROM JQUARTZ_FIRED_TRIGGERS WHERE ENTRY_ID = 'NON_CLUSTERED1510126796298'
```
可以看到，在两条插入语句之后，根本没有更新语句。此时的我已经开始怀疑人生了  
难道是网络错误，但是mysql返回的结果affect_rows确实是等于1，代表执行了呀。  
然后超哥说，上 `tcpdump`

好，上就上，开始用tcpdump抓包web服务器和mysql服务器的网络通信，过滤了IP和端口，截取一次请求的数据，导出来，放到Wireshark里面查看分析。  
结果出来了，
1. web服务器发送了那两个查询语句
2. mysql返回成功了！！！（分别返回了受影响行数1） 

超哥望着我，说可能是触发了mysql的什么bug，我。。。。还说mysql5.5 貌似bug挺多的。  
我，，，我就执行两个更新表语句，就触发bug了。这个bug太容易触发了吧，我真的不能相信。但是当时已经8点多了，我和超哥就都回去了，决定明天再来查查这个问题。

第二天过来，认真思考了一下，感觉可能是找错了方向。用全局眼光看问题。整体是在下单完成后执行的更新，难道下单这一块有问题？这里除了用了一个mysql事务的sql。好像和别的地方也没什么不一样。然后开始重新看下单的代码，涉及到mysql的地方，看到事务处理，果断把事务的代码全删了，裸执行sql测试，然后尝试请求，，，就成功了！

其实里面事务的操作只有三条语句
```
// $dao 是数据库基类，这里实际调用 mysqli 扩展里的函数
$dao->autocommit(false);    # 关闭 auto commit 
$dao->commit();             # 提交 commit
$dao->rollback();           # 回滚 rollback
```
相关一共就这三条语句，熟悉mysql事务的人应该已经看出问题。  
我先解释一下上面三条语句  
1. $dao->autocommit(false);
```
mysql 默认是开启事务的，默认情况下每执行一条语句就相当于提交一个事务，打开autocommit之后，会自动提交这个事务。我们想要一次执行多条语句，作为一个整体的事物的话，就要先把autocommit关掉，然后自己手动commit，这样多条sql可以作为一个整体提交或回滚。所以第一条语句 先关闭autocommit
```
2. $dao->commit(); 
```
这是在多条语句执行完之后，并且数据库返回成功，我们就手动commit一下这个事务
```
3. $dao->rollback();
```
如果多条语句钟中有执行失败的，则判断执行rollback方法，回滚所有的操作
```
好了，上面这三条语句放在这里是完全没问题的，之前的业务也证明它没毛病！

但是我们加了新的业务，想要在后面执行其他的sql语句时，由于autocommit被关闭，后面的sql将不会被提交，就造成一个很奇怪的现象，mysql其实并没有将这个操作commit，但是向客户端返回成功了！  
知道了问题所在，解决也很容易了。在订单最后的逻辑里面，重新把auto commit打开：
```
$dao->autocommit(true);
```
至此填完此坑，耗时近一天。  
本着知(zhong)识(sheng)分(zhui)享(ze)的原则, 找到了原来写下这段代码的同学。其实我很想吐槽一下，你的代码可以不要一句写那么长吗！  
![QQ](/assets/post-images/autocommit/0D0653B8-EFE0-46D3-A495-519374D89D63.png)