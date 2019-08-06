#关于ANR:
[anr详细分析](https://blog.csdn.net/tabactivity/article/details/52945343 )
## 
	ANR(Application Not Responding)，应用程序无响应，简单一个定义，却涵盖了很多Android系统的设计思想。
	首先，ANR属于应用程序的范畴，这不同于SNR(System Not Respoding)，SNR反映的问题是系统进程(system_server)失去了响应能力，而ANR明确将问题圈定在应用程序。 SNR由Watchdog机制保证，具体可以查阅Watchdog机制以及问题分析; ANR由消息处理机制保证，Android在系统层实现了一套精密的机制来发现ANR，核心原理是消息调度和超时处理。

	其次，ANR机制主体实现在系统层。所有与ANR相关的消息，都会经过系统进程(system_server)调度，然后派发到应用进程完成对消息的实际处理，同时，系统进程设计了不同的超时限制来跟踪消息的处理。 一旦应用程序处理消息不当，超时限制就起作用了，它收集一些系统状态，譬如CPU/IO使用情况、进程函数调用栈，
	并且报告用户有进程无响应了(ANR对话框)。

	然后，ANR问题本质是一个性能问题。ANR机制实际上对应用程序主线程的限制，要求主线程在限定的时间内处理完一些最常见的操作(启动服务、处理广播、处理输入)， 
	如果处理超时，则认为主线程已经失去了响应其他操作的能力。主线程中的耗时操作，譬如密集CPU运算、大量IO、复杂界面布局等，都会降低应用程序的响应能力。

	最后，部分ANR问题是很难分析的，有时候由于系统底层的一些影响，导致消息调度失败，出现问题的场景又难以复现。 这类ANR问题往往需要花费大量的时间去了解系统的一些行为，超出了ANR机制本身的范畴。
###ANR常见原因：
    1、Activity的UI在5秒内没有响应输入事件（例如，按键按下，屏幕触摸）–主要类型
		一个正常的输入事件会经过从outboundQueue挪到waitQueue的过程，表示消息已经派发出去;再经过从waitQueue中移除的过程，表示消息已经被窗口接收。 InputDispatcher作为中枢，不停地在递送着输入事件，当一个事件无法得到处理的时候，InputDispatcher的策略是放弃掉处理不过来的事件，
		并发出通知(这个通知机制就是ANR)，继续进行下一轮消息的处理

		在InputDispatcher派发输入事件时，会寻找接收事件的窗口，如果无法正常派发，则可能会导致当前需要派发的事件超时(默认是5秒)。 Native层发现超时了，
		会通知Java层，Java层经过一些处理后，会反馈给Native层，是继续等待还是丢弃当前派发的事件。

		extra:
			对于InputDispatcher而言，每注册一个InputChannel都被视为一个Connection，通过文件描述符来区别。InputDispatcher是一个消息处理循环，当有新的Connection时，就需要唤醒消息循环队列进行处理。
			InputDispatcherThread是一个线程，它处理一次消息的派发,输入事件作为一个消息，需要排队等待派发，每一个Connection都维护两个队列：
			outboundQueue: 等待发送给窗口的事件。每一个新消息到来，都会先进入到此队列
			waitQueue: 已经发送给窗口的事件
			publishKeyEvent完成后，表示事件已经派发了，就将事件从outboundQueue挪到了waitQueue
			事件经过这么一轮处理，就算是从InputDispatcher派发出去了，但事件是不是被窗口收到了，还需要等待接收方的“finished”通知。 在向InputDispatcher注册InputChannel的时候，同时会注册一个回调函数handleReceiveCallback():
			当收到的status为OK时，会调用finishDispatchCycleLocked()来完成一个消息的处理：

			InputDispatcher::finishDispatchCycleLocked()
			└── InputDispatcher::onDispatchCycleFinishedLocked()
			    └── InputDispatcher::doDispatchCycleFinishedLockedInterruptible()
			        └── InputDispatcher::startDispatchCycleLocked()
			调用到doDispatchCycleFinishedLockedInterruptible()方法时，会将已经成功派发的消息从waitQueue中移除， 进一步调用会startDispatchCycleLocked开始派发新的事件。

    2、BroadcastReceiver在10秒内没有执行完毕
		AMS维护着广播队列BroadcastQueue，AMS线程不断从队列中取出消息进行调度，完成广播消息的派发。 在派发“串行广播消息”时，会抛出一个定时消息BROADCAST_TIMEOUT_MSG，在广播接收器处理完毕后，AMS会将定时消息清除。 如果BROADCAST_TIMEOUT_MSG得到了响应，就会判断是否广播消息处理超时，
		最终通知ANR的发生。
    3、Service在特定时间内（20秒内）无法处理完成–小概率类型
		Service的ANR机制：通过定时消息跟踪Service的运行，当定时消息被响应时，说明Service还没有运行完成，这就意味着Service ANR。
		

	ANR的报告机制是通过AMS.appNotResponding()完成的，Broadcast和InputEvent类型的ANR最终也都会调用这个方法.

	Service和Broadcast都是由AMS调度，利用Handler和Looper，设计了一个TIMEOUT消息交由AMS线程来处理，整个超时机制的实现都是在Java层； 
	InputEvent由InputDispatcher调度，待处理的输入事件都会进入队列中等待，设计了一个等待超时的判断，超时机制的实现在Native层。

###为了避免ANR需要注意的点：
    1、主线程频繁进行IO操作，比如读写文件或者数据库；
    2、硬件操作如进行调用照相机或者录音等操作；
    3、多线程操作的死锁，导致主线程等待超时；
    4、主线程操作调用join()方法、sleep()方法或者wait()方法；
    5、system server中发生WatchDog ANR；
    6、service binder的数量达到上限。

##ANR样例：
   DALVIKTHREADS :
	(mutexes: tll=0 tsl=0 tscl=0 ghl=0 hwl=0 hwll=0)  
	"main" prio=5 tid=1 Sleeping
  | group="main" sCount=1 dsCount=0 obj=0x73f11000 self=0xf3c25800
  | sysTid=2957 nice=0 cgrp=default sched=0/0 handle=0xf7770ea0
  | state=S schedstat=( 107710942 40533261 131 ) utm=4 stm=6 core=2 HZ=100
  | stack=0xff49d000-0xff49f000 stackSize=8MB
  | heldmutexes=
  atjava.lang.Thread.sleep!(Native method)
  - sleepingon <0x31fd6f5d> (a java.lang.Object)
  atjava.lang.Thread.sleep(Thread.java:1031)
  - locked <0x31fd6f5d> (a java.lang.Object)
  atjava.lang.Thread.sleep(Thread.java:985)
  atcom.sunny.demo.MainActivity.startMethod(MainActivity.java:21)
  atjava.lang.reflect.Method.invoke!(Native method)
  atjava.lang.reflect.Method.invoke(Method.java:372)
  atandroid.view.View$1.onClick(View.java:4015)
  atandroid.view.View.performClick(View.java:4780)
  atandroid.view.View$PerformClick.run(View.java:19866)
  atandroid.os.Handler.handleCallback(Handler.java:739)
  atandroid.os.Handler.dispatchMessage(Handler.java:95)
  atandroid.os.Looper.loop(Looper.java:135)
  atandroid.app.ActivityThread.main(ActivityThread.java:5254)
  atjava.lang.reflect.Method.invoke!(Native method)
  atjava.lang.reflect.Method.invoke(Method.java:372)
  atcom.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:903)
  atcom.android.internal.os.ZygoteInit.main(ZygoteInit.java:698)

###ANR日志的分析：
    第1行是固定头，指明下面都是当前运行的dvm thread：“DALVIK THREADS”；
    第2行输出的是改进程中各线程互斥量的值，有些手机上面可能没有这一行日志信息；
    第3行输出的是线程名字(“main”)，线程优先级(“prio=5”)，线程id(“tid=1”)，线程状态(Sleeping),比较常见的状态还有Native、Waiting；
    第4行分别是线程所处的线程组 （“main”），线程被正常挂起的次处（“sCount=1”），线程因调试而挂起次数（”dsCount=0“），当前线程所关联的java线程对象（”obj=0x73f11000“）以及该线程本身的地址（“0xf3c25800”）；
    第5行 显示线程调度信息，分别是该线程在linux系统下得本地线程id （“ sysTid=2957”），线程的调度有优先级（“nice=0”），调度策略（sched=0/0），优先组属（“cgrp=default”）以及 处理函数地址（“handle=0xf7770ea0”）；
    第6行 显示更多该线程当前上下文，分别是调度状态（从 /proc/[pid]/task/[tid]/schedstat读出）（“schedstat=( 107710942 40533261 131 )”），以及该线程运行信息 ，它们是线程用户态下使用的时间值(单位是jiffies）（“utm=4”）， 内核态下得调度时间值（“stm=6”），以及最后运行改线程的cup标识（“core=2”）；
    第7行表示线程栈的地址（“stack=0xff49d000-0xff49f000”）以及栈大小（“stackSize=8MB”）；
    后面是线程的调用栈信息，也是分析ANR的核心所在。

###如何避免ANR:
    1、避免在主线程进行复杂耗时的操作，特别是文件读取或者数据库操作；
    2、避免频繁实时更新UI；
    3、BroadCastReceiver 要进行复杂操作的的时候，可以在onReceive()方法中启动一个Service来处理；
    4、避免在IntentReceiver里启动一个Activity，因为它会创建一个新的画面，并从当前用户正在运行的程序上抢夺焦点。
    5、如果你的应用程序在响应Intent广播时需要向用户展示什么，你应该使用Notification Manager来实现。
    6、在设计及代码编写阶段避免出现出现同步/死锁或者错误处理不恰当等情况。