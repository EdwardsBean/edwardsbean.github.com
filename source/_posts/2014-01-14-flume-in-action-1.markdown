---
layout: post
title: "Flume in action(1)"
date: 2014-01-14 17:10:07 +0800
comments: true
categories: 
---

## 使用Flume Log4j Appender正确的姿势
我们使用Flume-ng的LoadBalancingLog4jAppender，将线上服务的日志实时传输到日志服务器，转交给告警系统和HDFS做存储。  
FLume的Log4j Appender必须使用Log4j的异步加载器，否则一旦日志服务器挂掉，将会导致应用服务器宕机。  
## 使用过程中的坑
####问题1： Flume Log4j使用异步加载器，日志服务器宕机情况导致业务系统阻塞  
在阅读了Flume的RPC源码以及LoadBalancingLog4jAppender的实现之后，发现问题原来在Log4j的异步加载器AsyncAppender。异步加载器的原理见[这里](http://www.blogbus.com/blackgu-logs/163664173.html)  
根本原因是、日志服务器宕机导致消费者消费能力不足，缓冲区满的情况下，AsyncAppender会阻塞程序。设置Blocking=false之后就可以了。
####问题2：Flume Log4j发送大量链接异常日志
当其中一台日志服务器宕机，其他的日志服务器就会不停的接收到链接异常的日志。明显是重连的时间间隔太短。在LoadBalancingRpcClient中，
```
    while (it.hasNext()) {
      HostInfo host = it.next();
      try {
        RpcClient client = getClient(host);
        client.append(event); 
        eventSent = true;
        break;
      } catch (Exception ex) {
        selector.informFailure(host); //宕机情况标志该主机异常
        LOGGER.warn("Failed to send event to host " + host, ex);
      }
    }
```
Flume默认不启用back off,也就是说selector.informFailure(host)这行代码完全没用。简直坑爹。OrderSelector.java:
```
  public void informFailure(T failedObject) {
    //If there is no backoff this method is a no-op.
    if (!shouldBackOff) {
      return;
    }
    //将该主机暂时移除可用主机列表
    ...
    
```
所以解决办法：配置max back off
####问题3：Flume Log4j失败重连策略异常
问题体现在，设置了max back off,重连时间居然一直是2000ms,看了一下它的算法，指数退避算法。在OrderSelector.java的informFailure函数中。
```
  public void informFailure(T failedObject) {
    //If there is no backoff this method is a no-op.
    if (!shouldBackOff) {
      return;
    }
    FailureState state = stateMap.get(failedObject);
    long now = System.currentTimeMillis();
    long delta = now - state.lastFail;
    long lastBackoffLength = Math.min(maxTimeout, 1000 * (1 << state.sequentialFails));
    long allowableDiff = lastBackoffLength + CONSIDER_SEQUENTIAL_RANGE;
    if (allowableDiff > delta) {
      if (state.sequentialFails < EXP_BACKOFF_COUNTER_LIMIT) {
        state.sequentialFails++;
      }
    } else {
      state.sequentialFails = 1;
    }
    state.lastFail = now;
    //Depending on the number of sequential failures this component had, delay
    //its restore time. Each time it fails, delay the restore by 1000 ms,
    //until the maxTimeOut is reached.
    state.restoreTime = now + Math.min(maxTimeout, 1000 * (1 << state.sequentialFails));
  }
```
最后生成的restoreTime即下一次进行重试的时间。我没有去设置avro connect time out 和request time out，默认都是20s,应该算是偏长了。根据他的算法，delta永远是大于40s，但是allowableDiff却一直是3s,4s.所以我直接改了判定条件，allowableDiff < delta,之后就正常。但是还存在一个问题，sequentialFails并不会在一段时间后reset.
###问题4：Log4j异步加载器丢失日志数据
AsyncAppender默认缓冲区大小128，满了之后会丢失数据。调大缓冲区，avro connect time out 和request time out也得适当调一下