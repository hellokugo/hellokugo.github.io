title: MQTT源码解析之connect
date: 2016/12/20 20：40

categories:
- Android
tags:
- Android，MQTT
---

# 说在前面

前段时间项目组在搞 [IM](http://blog.csdn.net/DreamTww/article/details/4632174)，前期主要是对当前IM类似QQ、微信和环信等通信协议的采用分析，基本确定是在XMPP和MQTT二者选一。XMPP应用较为广泛稳定，但是如果应用在移动端实在是折腾不起，流量大，自然耗电量也会多。<!-- more -->相比之下，初衷使用在推送方面的MQTT协议简直是为移动端的痛点应运而生。 [参考](http://xiangwangfeng.com/2015/05/20/%E7%A7%BB%E5%8A%A8IM%E5%BC%80%E5%8F%91%E9%82%A3%E4%BA%9B%E4%BA%8B/)

# 说些什么

本文不是要详细展开对MQTT协议的解析，而是主要从github上的 [paho demo](https://github.com/sandro-k/org.eclipse.paho.android.service.sample) 来对MQTT内部实现connect进行流程的梳理，结合其 [官方文档](http://public.dhe.ibm.com/software/dw/webservices/ws-mqtt/mqtt-v3r1.html) 对协议格式的了解，相信读者能拨开云雾，了解其实现机制。另外，推荐下国人对MQTT理解写的 [系列文章](http://www.blogjava.net/yongboy/category/54835.html)。注意的是，要跑起demo需要搭建测试服务器作订阅发送，建议采用mosquitto作连接服务端供本地测试， [参考地址](https://code.google.com/archive/p/mosquitto-mqtt-client-android/)

# Read Code

## Prepare

假如demo能成功在手机端跑起，并且开启mosquitto，开始介绍其connect流程，跟进代码到ClientConnections类中的connectAction方法：

![](http://i.imgur.com/mXz6h2H.png)

获取到新建连接的cid、uri和port信息，注意，这里是用本地的mosquitto作测试连接，所以uri通过ipconfig获取到的ip地址填入，port默认是1883（当然，你可以作修改），cid可以随意。

其中还有其他一些参数（诸如ssl、qos和keepalive等）由于本篇主要介绍的是conncet所以不展开细说。ok，接着往下看：

![](http://i.imgur.com/WzEHxZO.png)

![](http://i.imgur.com/RJIgpES.png)

这里要对回调进行区分，以上截图第一个回调MqttCallbackHandler主要是接收服务端push过来的信息回调；而下图的callback继承于ActionListener，顾名思义，是发起本次“动作”的回调结果，连接是一个动作，订阅是一个动作，客户端push信息也是一个动作，每次动作的发起都会注册回调返回结果。区分好上述两个回调后，开始走进connect操作的核心。

## MqttService

connect前还需要做一个操作，就是区分每一次连接，这里是用MqttTokenAndroid来保存当前连接的callback。我们知道，连接过程的时间并不是我们所能控制的，所以这里开启了一个service来进行后面的操作。这里需要说明一点的是，在startService后进入到MqttService的onStartCommand方法中，注册了一个广播：

![](http://i.imgur.com/WlIsDzx.png)

该广播主要是检测手机网络的变化而进行重连或断开连接的提示操作。ok，在开启startService和bindService后注册了另外一个广播：

![](http://i.imgur.com/NNDEpwq.png)

这个广播至关重要，因为它是接收服务端信息的桥梁，action是MqttServiceConstants.CALLBACK_TO_ACTIVITY，在后面会用到。

## connect

连接service成功后回调至onServiceConnected中做doConnect()方法，跟进代码，有这样一句：

![](http://i.imgur.com/IKUDaul.png)

这里巧妙地用一个以递增num作key、token作value的map来保存映射每次的连接操作。之后做mqttService.connect()。

跟到MqttConnection的connect方法，设置一个回调的bundle，action指定为CONNECT_ACTION，并把传进来的token带上，这里多设置了一层代理的IMqttActionListener，把上述设置的bundle带上。然后就new出一个MqttAsyncClient，并绑定上述设置的IMqttActionListener作为回调，然后再做connect()。

## tcp / ssl

跟进MqttAsyncClient类中的connect()方法，在进行一系列的当前连接状态判断之后，有以下一句代码：

![](http://i.imgur.com/OXZzldo.png)

这里面是干嘛的呢？跟进去看下：

![](http://i.imgur.com/03bxLCO.png)

原来是设置不同连接的端口和uri地址，这里是根据connect前是否设置ssl安全访问模式去设定的，默认ssl为false，即tcp连接。

## connect protocol infomation

跟到这里，发现复杂的代码总是设置各种代码和多层回调，给人雾里看花的感觉，也许这就是设计模式的魅力。别急，接着往下看，跟到ClientComms类的connect()方法。看到这里：

![](http://i.imgur.com/b4rdqKi.png)

在MqttConnect的构造方法中第一句代码是 

	super(MqttWireMessage.MESSAGE_TYPE_CONNECT);
终于看到设置connect协议的地方了，为什么这么说？了解mqtt头部信息的读者应该对以下的图并不陌生：

![](http://i.imgur.com/ynGryag.png)

要想进一步了解connect协议字段，可以参考文章开篇推荐的国人总结的系列文章中有提及到。 [送上链接](http://www.blogjava.net/yongboy/archive/2014/02/09/409630.html)

## connect Thread

看到设置connect协议了，应该离真正连接的代码不远了。继续往下看：

![](http://i.imgur.com/3cevn7V.png)

咦，这里开了一个线程，干嘛的呢？看下ConnectBG的run()方法：

![](http://i.imgur.com/ucJCDSt.png)

接下来，打算分三部分来介绍。

### socket 

NetworkModule是一个接口，真正实现的是在上述tcp/ssl介绍部分的TCPNetworkModule/LocalNetworkModule。这里以TCPNetworkModule为例，上代码：

![](http://i.imgur.com/ujIbCD6.png)

哎呀，原来是用到socket连接，这下终于看到真面目了。这一步是打开了socket。

### run Threads

网络连接了，必然是开始数据的发送和接收处理。果不其然，接下来看到的是三个线程，分别是CommsReceiver、CommsSender和CommsCallback。
下面直接看下这三个线程的run方法：

***CommsReceiver***：

![](http://i.imgur.com/lb0DTw3.png)

该线程用 in.available() > 0 处理接收消息，如果没有接收到消息则一直处于阻塞状态；

***CommsSender***：

![](http://i.imgur.com/bG7FV1c.png)

通过clientState.get()获取到发送消息，点进去get()方法看下：

![](http://i.imgur.com/nc5mpmS.png)

这里定义pendingMessages和pendingFlows两个容器来处理不同的发送消息类型，并释放queueLock锁，queueLock.wait()一直阻塞等待被唤起。

***CommsCallback***：

![](http://i.imgur.com/Agf4Pis.png)

和CommsSender类似，通过释放workAvailable锁，workAvailable.wait()一直阻塞等待被唤起。

### send connect request

看到上面介绍的三个线程均是阻塞状，接下来肯定是需要唤醒某个线程做发送操作了。跟进internalSend()方法：

![](http://i.imgur.com/UbJIeJv.png)

看到这一步验证了我们的猜想，被唤起的是CommsSender这个线程：

![](http://i.imgur.com/CzzhoTL.png)

最终通过out.write()和out.flush()把connect消息发送出去。

## receive connectAck

把coneect消息发送出去只是成功一半，接收到服务端的connectAck才算是connect成功。So，服务端发送过来的消息在哪里接收呢？CommsReceiver在获取到消息则不在阻塞，第一步先解析获取到数据类型：

![](http://i.imgur.com/rEuV1ov.png)

如果阅读过推荐的解析MQTT协议文章的读者在这里应该理解不难，跟进createWireMessage()方法，由于收到的是connectAck，所以会走到这里：

![](http://i.imgur.com/TVppG3e.png)

建立并返回的是一个MqttAck对象。继续返回到CommsReceiver：

![](http://i.imgur.com/FQSP8Nf.png)

跟进clientState.notifyReceivedAck((MqttAck)message)，走到这里：

![](http://i.imgur.com/iEmslY9.png)

获取rc状态码，等于0则会成功，这里面connected()方法是一个关键，主要是做心跳连接，这个后面再细讲；跟进notifyResult() 方法：

![](http://i.imgur.com/pbVUvhD.png)

![](http://i.imgur.com/58Q2QiK.png)

哎呀，跟到最后，原来是为了唤醒CommsCallback这个线程。Then，跟到CommsCallback：

![](http://i.imgur.com/irBkubN.png)

接着跟进去handleActionComplete()方法：

![](http://i.imgur.com/BfpZpiu.png)

![](http://i.imgur.com/5gCV2MN.png)

至此，线程已经走完，剩下的就是之前设置下来的回调通知。这个相信读者可以自行往下阅读。

# 说到最后

整个connect流程走下来还是很清晰可见的，后续会继续分析心跳ping、订阅和publish等流程，其实过程都是大同小异，都是基于以上三个线程去进行。所以，熟悉connect流程对之后的分析帮助很大。