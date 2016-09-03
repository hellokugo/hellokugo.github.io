title: 简析Android组件安全
date: 2016/6/14 17：57

categories:
- Mobile
tags:
- android
---

# 前言 #
前段时间在学习github上DroidPlgin插件的时候接触到不少专业术语，诸如预先占坑、Activity劫持和Hook等，一开始看还真是感觉雾里看花。在查阅资料的过程中发现了国人出版的《Android软件安全与逆向分析》，在理解Activity劫持的过程中引申了Android四大组件的安全。知识点不难却感觉蛮有趣，故分享之。<!-- more -->

## Activity ##
Android组件是用户唯一能够看见的组件，作为软件所有功能的显示载体，其安全问题是最应该受到关注的
### 权限攻击 ###
正如Android开发文档中所说的，Android系统组件在制定Intent过滤器（intent-filter）后，默认是可以被外部程序访问的。这就很意味着很容易被其他程序进行串谋攻击，关于串谋攻击，在分享最后再做一个简单的解析。现在问题是，如何防止Activity被外部使用呢？

Android所有组件声明时可以通过指定android:exported属性值为false，来设置组件不能被外部程序调用。这里的外部程序是指签名不同、用户ID不同的程序，签名相同且用户ID相同的程序在执行时共享同一个进程空间，彼此之间是没有组件访问限制的。如果希望Activity能够被特定的程序访问，就不能用android:exported属性了，可以使用android:permission属性来指定一个权限字符串，声明例子如下：


    <Activity android:name=".MyActivity"
           android:permission="com.test.permission.MyActivity">
           <intent-filter>
               <action android:name="com.test.action"></action>
           </intent-filter>
    </Activity>

这样声明的Activity在被调用时，Android系统就会检查调用者是否具有com.test.permission.MyActivity权限，如果不具备就会触发一个SecurityException安全异常。要想启动该Activity必须在AndroidManifest.xml文件中加入以下声明：
    
    <uses-permission android:name="com.test.permission.MyActivity">

### Activity劫持 ###
当用户安装了带有Activity劫持功能的恶意程序后，恶意程序会遍历系统中运行的程序，当检测到需要劫持的Activity（通常是网银或其他网络程序的等登陆界面）在前台运行时，恶意程序就会启动一个带FLAG\_ACTIVITY\_NEW\_TASK标志的钓鱼式Activity覆盖正常的Activity，从而欺骗输入用户名或密码信息，当用户输入完信息后，程序就会将信息发送到指定的网址或邮箱，然后切换到正常的Activity中。这就是Activity劫持原理。从受影响的角度看，Activity属于用户层安全，程序员无法控制。

下面做一个简单的劫持例子。实例可以对多个进程进行劫持，它在启动时启动了一个Hack服务，Hack服务创建了一个定时器，定时器每隔2秒就检测一次系统正在运行的进程，判断前台运行的进程是否匹配劫持的进程列表中，如果匹配就进行劫持：

    private TimeTask mTask = new TimeTask(){
    
        @Override
        public void run(){
         	ActivityManager am = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
            List<RunningAppProcessInfo> infos = am.getRunningAppProcesses();
            //枚举进程
            for(RunningAppProcessInfo psinfo:infos){
                if(psinfo.importance == RunningAppProcessInfo.IMPORTANCE_FOREGROUND){ //前台进程
                 	if(hackList.contains(psinfo.processName)){
 	                    Intent intent = new Intent(getBaseContent(),HackerActivity.class);
                        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)；
                        intent.putExtra("processname",psinfo.processName);
                        getApplication().startActivity(intent);//启动伪造的Activity
                    }
                 }
            }
        }

    }

Activity劫持不需要在AndroidManifest.xml中声明任何权限就可以实现，一般的防病毒软件也无法检测，手机用户更是防不胜防。不过有一种方法可以判断当前Activity是否被劫持：长按Home键不放，系统显示最近运行过的程序列表。在本例中，HackerActvity显示在最前面，很显然这个程序就有劫持Actvity的嫌疑。

Unfortunately，这种检测Activity劫持的方法并不是任何时候都有效的，假如在声明Activity时，设置android:excludeFromRecents的值为true，程序在运行时就不会显示在最近运行过的程序列表中，上面的检测方法也就只能hehe了。

## Broadcast Receiver ##
Broadcast Receiver用于处理发送和接收广播，这里分为发送安全和接收安全。接收广播涉及串谋攻击，这里先不作介绍，下面主要介绍发送广播安全。先来一段熟悉的代码:

	Intent intent = new Intent();
	intent.setAction("com.test.broadcast");
	intent.putExtra("data",Math.random());
	sendBroadcast(intent);

这段代码发送了一个Action为com.test.broadcast的广播。我们都知道，Android系统提供了两种广播发送方法，分别是sendBroadcast()和sendOrderedBroadcast()。sendBroadcast()用于发送无序广播，该广播能够被所有广播接受者接收，并且不能被abortBroadcast()终止，sendOrderedBroadcast()用于发送有序广播，该广播被优先级高的广播接收者有限接收，然后依次向下传递，优先级高的广播接收者可以篡改广播，或者调用abortBroadcast()终止广播。广播优先级响应的计算方法是：动态注册的广播接收者比静态广播接收者的优先级高，静态广播接收者的优先级根据设置的android:priority属性的数值来决定，数值越大，优先级越高。

从上面分析可以得知，假如Hacker动态注册一个Action为com.test.broadcast的广播接收者，并且拥有最高的优先级，上述程序假如使用sendBroadcast()发送广播，Hacker的确无法通过abortBroadcast()终止，但可以优先响应该实例发送的广播；但是假如上述程序使用sendOrderedBroadcast()发送，很有可能BroadcastReceiver实例就永远无法收到自己发送的广播。

当然，上述问题也是可以避免的。在发送广播时通过Intent指定具体要发送到的Android组件或类，广播就永远只能被本实例指定的类所接收，如：

	intent,setClass(MainActivity.this,test.class);

## Service ##
Service组件是Android系统中的后台进程，主要的功能是在后台进行一些耗时的操作。与其他的Android组件一样，当声明Service时指定了Intent过滤器，该Service默认就可以被外部访问。可以访问的方法有：

>startService():启动服务，可以被用来实现串谋攻击。

>bindService():绑定服务，可以被用来实现串谋攻击。

>stopService():停止服务，对程序功能进行恶意破坏。

对于恶意的stopService，它破解程序的执行环境，直接影响到程序的正常运行。要想杜绝Service组件被人恶意的启动或者停止，就需要使用Android系统的权限机制来对调用者进行控制。如果Service组件不想被程序外的其他组件访问，可以直接设置它的android:exported属性为false，如果是同一作者的多个程序共享使用该服务，则可以使用自定义的权限，这在Activity的权限攻击已经介绍过，这里就不铺开来说了。

## Content Provider ##
Content Provider 用于程序之间数据交换。Android系统中，每个应用的数据库、文件、资源等信息都是私有的，其他程序无法访问，如果想要访问这些数据，就必须提供一种程序之间数据的访问机制，这就是Content Provider的由来。一个典型的Content Provider声明如下：

	<provider
		android:name="com.test.providerhehe"
		android:authorities="cpm.test.heheprovider"
		android:readPermission="droider.permission.FILE_READ"
		android:writePermission="droider.permission.FILE_WRITE"
	>

Content Provider提供了insert()、delete()、update()、query()等操作，其中执行query()查询操作时会执行读权限android:readPermission检查，其他的操作会执行写权限android:writePermission检查，权限检查失败时会抛出SecurityException异常。对于很多开发人员来说，在声明Content Provider 时几乎从来不使用这两个权限，这导致了串谋攻击发生的可能。部分网络软件开发商使用Content Provider来实现软件登陆、用户密码修改等敏感度极高的操作，然而声明的Content Provider却没有权限控制，这使得一些恶意软件无需任何权限就可以获取用户的敏感信息。

## 串谋权限攻击 ##
Android程序中资源的访问包括使用Framework提供的功能与访问其他程序的组件，前者是通过系统提供的权限机制进行控制的，后者是通过自定义权限控制的。正常情况下，没有声明特定的访问权限，就无法访问这些资源。但通过其他程序中可访问的Android组件，就有可能突破这种访问控制，从而提升程序本身的权限，这种权限提升的攻击方式在这称为串谋权限攻击。

![串谋权限攻击示意图](https://cloud.githubusercontent.com/assets/10484766/11139875/7c3829ba-8a0f-11e5-838c-83064d88a724.png)

这里可以模拟一个场景，有一个下载管理程序实例DownloadManager，有一个TextView输入想要下载的文件URL，点击下载按钮即开始下载，下载下来的文件默认保存在SD卡上。当然，该实例拥有下载文件和保存SD卡的权利：

	<uses-permission androidLname="android.permission.WRITE_EXTERNAL_STORAGE/">
	<uses-permission android:name="android.permission.INTERNET">

下载文件的功能是通过接收下载请求广播，然后在下载广播接收者中完成的，对应的广播接收者在AndroidManifest.xml中声明：

	<receiver android:name=".DownloadReceiver">
		<intent-filter>
			<action android:name="com.test.download"></action>
		</intent-filter>
	</receiver>

DownloadReceiver响应Action为"com.test.download"的广播，然后访问Intent指定的URL地址去下载文件，相应的广播响应代码如下：

	public void onReceive(Context context,Intent intent){
		if(intent.getAction().equals("com.test.download")){
			String url = intent.getExtra().getString("url");
			String filename = intent.getExtra().getString("filename");
			Toast.makeText(context,url,Toast.LENGTH_SHORT).show();
			try{
				downloadFile(url,filename);
			}catch(IOException e){
				e.printStackTrace();
			}
		}
	}

从上面代码可以清楚地看到，下载的文件时通过url传递过来而保存的文件名则是通过filename传递。接下来就是我们的攻击程序HackerDownloader，它什么权限都没有，只有一段发送文件下载请求的代码：

	btn.setOnClickListener(new OnClickListener(){
		@override
		public void onClick(){
			Intent intent = new Intent();//创建Intent对象
			intent.setAction("com.test.download");
			intent.putExtra("url","http://********");//要下载文件url地址
			String fileName = "test.txt";
			intent.putExtra("filename",fileName);
			sendBroadcast(intent);
		}
	});

当HackerDownloader点击下载按钮以后，DownloadReceiver收到广播后就会开始下载相对应的文件并保存到指定目录下。至此，一次完美的串谋攻击就完成了。

