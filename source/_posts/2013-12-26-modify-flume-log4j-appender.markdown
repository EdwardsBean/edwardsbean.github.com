---
layout: post
title: "修改Flume Log4j Appender"
date: 2013-12-26 11:20:47 +0800
comments: true
categories: Flume
---



##自定义Log4j Appender
要修改Flume Log4j Appender的实现，我们先了解一下Log4j Appender是如何自定义的。

自定义log4j appender需要继承log4j公共的基类：AppenderSkeleton

- 打印日志核心方法：abstract protected void append(LoggingEvent event);
- 初始化加载资源：public void activateOptions()，默认实现为空
- 释放资源：public void close()
- 是否需要按格式输出文本：public boolean requiresLayout()
- 正常情况下我们只需要覆盖append方法即可。然后就可以在log4j中使用了

Demo:

	1. import org.apache.log4j.AppenderSkeleton;  
	2. import org.apache.log4j.spi.LoggingEvent;  
	3.   
	4. public class HelloAppender extends AppenderSkeleton {  
	5.   
	6.     private String account ;  
	7.       
	8.     @Override  
	9.     protected void append(LoggingEvent event) {  
	10.         System.out.println("Hello, " + account + " : "+ event.getMessage());  
	11.     }  
	12.   
	13.     @Override  
	14.     public void close() {  
	15.         // TODO Auto-generated method stub  
	16.   
	17.     }  
	18.   
	19.     @Override  
	20.     public boolean requiresLayout() {  
	21.         // TODO Auto-generated method stub  
	22.         return false;  
	23.     }  
	24.   
	25.     public String getAccount() {  
	26.         return account;  
	27.     }  
	28.   
	29.     public void setAccount(String account) {  
	30.         this.account = account;  
	31.     }  
	32. }  
------------------

	1. public static void main(String[] args) {  
	2.     Log log = LogFactory.getLog("helloLog") ;  
	3.     log.info("I am ready.") ;  
	4. }  

*log4j.properties 配置*
```
log4j.logger.helloLog=INFO, hello

log4j.appender.hello=HelloAppender
log4j.appender.hello.account=World
```

执行main函数，输出结果
Hello, World : I am ready.

##修改FLume Log4j Appender 
### Event Header加入appname,hostname,logtype
由于hostname,logtype是固定不变的。所以直接写死在代码中。appname则用log4j.properties进行配置
- appname在log4j.properties中配置,添加：

```
#应用程序名
log4j.appender.flume.Appname = 91pc
```

- 在Log4jAppender类中添加：

```
private String appname;

public String getAppname() {
	return appname;
}

public void setAppname(String appname) {
	this.appname = appname;
}
```

- hostname,logtype在Log4jAppender类中添加

```
    //author:scy
    try {
		hdrs.put("hostname", InetAddress.getLocalHost()+"");
	} catch (UnknownHostException e1) {
	      String msg = "Cant't get localhost IP";
	      LogLog.error(msg);
	      if (unsafeMode) {
	        return;
	      }
	      throw new FlumeException(msg + " Exception follows.", e1);
	}
    //author:scy Need to alarm(>Info)
    if(event.getLevel().toInt() > 20000){
    	hdrs.put("logtype", "alarm");
    }else{
    	hdrs.put("logtype", "log4j");
    }
```

###Flume Log4j Appender添加log4j的异常详细信息
由于Flume Log4j Appender并没有将Log4j的错误异常栈详细信息封装到Event中，不利于我们的告警系统分析原因。
Log4jAppender.append()中添加如下：
```
    Event flumeEvent;
    Object message = event.getMessage();
    if (message instanceof GenericRecord) {
    ..
    } else {
      hdrs.put(Log4jAvroHeaders.MESSAGE_ENCODING.toString(), "UTF8");
      //按照log4j.properties配置格式化日志
      String msg = layout != null ? layout.format(event) : message.toString();
            
      //author:edwardsbean
      if(layout.ignoresThrowable()) {
          String[] s = event.getThrowableStrRep();
          if (s != null) {
    	int len = s.length;
    	for(int i = 0; i < len; i++) {
    		msg += s[i];
    		msg += Layout.LINE_SEP;
    	}
        }
      }
      
      flumeEvent = EventBuilder.withBody(msg, Charset.forName("UTF8"), hdrs);
    }
    try {
      rpcClient.append(flumeEvent);
```
日志接收端就可以接受到日志的详细信息：
```
 [x] Received '2013-12-26 10:38:08 user log detail
 [nd-PC2600/192.168.253.126] FATAL [com.xx.test.Main] java.lang.Exception: error detail
	at com.xx.test.Main.main(Main.java:35)
```
