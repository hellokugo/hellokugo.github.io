title: MQTT源码解析之心跳ping
date: 2017/2/1 14：18

categories:
- Android
tags:
- Android，MQTT，ping
---

# 前言

前面已经解析了MQTT的 [connect连接源码](https://hellokugo.github.io/2016/12/20/MQTT%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B9%8Bconnect/) ，接下来介绍MQTT心跳实现机制。<!-- more -->

# 实现原理

我们知道，手机客户端对流量和电量的要求极为苛刻。如果要保持MQTT的长连接就无法避免上述两个条件的限制。网上有很多检测网络是否连接的方法，这里从MQTT源码出发，找到一些相关的资料：

[android设备休眠](http://www.cnblogs.com/kobe8/p/3819305.html)

[Android手机的休眠状态](http://blog.csdn.net/berber78/article/details/46696675)

推荐这两篇文章，都是在阐述Android手机有AP和BP两个处理器。事实上，MQTT也是从这思路去考虑，提供两种方式去实现心跳ping，分别是TimerTask和AlarmManager，具体实现类看继承关系：

![](http://i.imgur.com/RhuZtuK.png)

深入查看MQTT的当前实现，是采用AlarmManager这种方法来实现心跳连接。这也是Android官方推荐的定时任务实现，有一篇文章做了这两种实现方式的相关测试 [Android定时任务详解](http://jellybins.github.io/2016/01/26/Android%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E8%AF%A6%E8%A7%A3/)。

# ping消息格式

因为ping消息是我们自定义用作判断连接的消息，所以这个消息尽可能避免给正常收发消息带来不必要的消耗，而mqtt对ping消息的格式设计是怎样的呢？我们可以看下MqttPingReq.java这个类：

![](http://i.imgur.com/bCTNRAX.png)

很简单，定义了MESSAGE_TYPE_PINGREQ的消息类型，同时注意到复写getVariableHeader方法返回一个0字节的数组，同时也没有复写父类的getPayload方法，这说明ping消息结构只是简单的包含头部信息，即只有两个字节，非常轻便。

# flow the code

## 定义AlarmManager

ok，我们开始看下具体实现代码。首先，我们上面说到，MQTT采用的是AlarmManager这种方式来实现ping，这个可以从MqttConnection类中connect方法中看到其初始化：

![](http://i.imgur.com/GsbTAF0.png)

那么，它究竟什么时候开始ping呢？正常的想，应该是connect成功之后就可以开始，那就是收到connectAck之后。在介绍connect时说过，接收服务端发送过来的代码实现在CommsReceiver类中的线程中，追踪下代码，果不其然，在收到connectAck后进行解析：

![](http://i.imgur.com/2b2xsAU.png)

这个connected()方法就是ping的逻辑。开始跟进去看看：

![](http://i.imgur.com/SEJQQOs.png)

这里的pingSender就是AlarmPingSender的实例，跟进AlarmPingSender的start方法：

![](http://i.imgur.com/FFrpFgh.png)

这里注册了一个alarmReceiver的广播，这个稍后解析；定义一个和alarmReceiver一样action的pendingIntent，在schedule方法中具体实现：

![](http://i.imgur.com/RRs2GTP.png)

逻辑比较简单，定义了AlarmManager，并且在delayInMilliseconds（当前时间 + delayInMilliseconds）后去发送广播到alarmReceiver。那么，这个delayInMilliseconds即comms.getKeepAlive()是怎么传入的呢？这个可以在连接前定义MqttConnectOptions时进行设置：

![](http://i.imgur.com/Zl7j7R6.png)

如果不设置，默认值是60s。

## alarmReceiver

上面介绍说了，AlarmManager会在delayTime后发送广播，alarmReceiver的具体实现是AlarmReceiver类的onReceive方法中：

![](http://i.imgur.com/3Z772yr.png)

这里面主要分三部分，第一部分是发送ping的关键代码逻辑，也包含判断当前网络是否ping失败和规划下次ping的时间；第二部分是获取PowerManager的wakelock（这个是针对ping返回callback处理的wakelock，尽管在设置alarmManager时已经有获取过wakelock），保持CPU运转；第三部分是在发送ping后等待收到服务端pingResponse释放wakelock以尽可能减少对手机电源的损耗。接下来介绍下第一部分。

## ping

![](http://i.imgur.com/t5a4GoO.png)

这里首先判断上一次ping是否失败，包括手机客户端和服务端的异常，这里说明一下，pingOutstanding是每次发送ping都会加1，收到pingResponse就减1；lastInboundActivity是代表上一次收到服务端消息的时间，相反地lastOutboundActivity则是上一次发送消息的时间；设置delta是因为代码使用System.currentTimeMillis()获取当前时间会有误差，所以尽可能加减delta值减少误差。

两种异常注释已经解释得很清楚了。第一种异常是客户端发送ping消息后，在距离上一次接受到服务端发送回来的消息时间大于keepAlive + delta 认为是服务端没有reply客户端的ping消息异常；第二种异常则是认为客户端发送的ping没有在两个keepAlive时间发出，此时抛出写ping异常。接下来是对当前时间判断是否需要发送ping或者动态规划下一次接收广播时间：

![](http://i.imgur.com/Gi7djvl.png)

首先判断此次keepAlive时间内客户端是否有发送或者接收信息，如果没有则发送ping消息，并把下次ping的时间设置成当初传入的keepAlive，这里的判断条件有两种情况，注释也说明得比较清晰；如果在此次keepAlive时间内客户端有收发信息（除ping消息外），则重新规划下次ping的时间，这里主要是以上一次客户端发送消息的时间为准，以设置全局的keepAlive和当前时间做一个差运算，time - lastOutboundActivity即离上一次客户端发送信息时间越长，则设置下次ping时间越短，这个也很好理解，如果当前客户端收发信息频繁，则没必要在短时间内自定义发送ping，此时应该认为断开连接的异常机会较小，应当减少自定义ping。这种动态规划ping时间很巧妙，尽可能保证可以检测出连接异常，同时也减少ping的频率和消耗。

## 释放WakeLock

在上述分析中，成功发送ping消息后，获取手机WakeLock并针对此次ping的返回释放WakeLock。过程和介绍connect流程基本一致，也是走CommsSender-->CommsReceiver-->CommsCallback这三个线程的run方法，具体的代码这里就不跟了。最终走到CommsCallback的handleActionComplete方法，执行fireActionEvent方法对ping保存的token进行回调。记住，无论成功与否，必须释放本次创建的WakeLock，避免不必要的消耗。

至此，mqtt的ping流程已经介绍完毕。