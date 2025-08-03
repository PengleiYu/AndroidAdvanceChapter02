# Chapter02

简介
======
该例子主要演示了如何通过关闭`FinalizerWatchdogDaemon`来减少`TimeoutException`的触发

需要注意的是，此种方法并不是去解决问题，而是为了避免上报异常采取的一种 hack 方案，并没有真正的解决引起 `finialize()` 超时的问题。

界面
======

![screen](screen.png)



操作步骤
======

1. 最好在模拟器下执行例子，因为各个手机设置的超时时长不同，不容易观看效果。
2. 点击`触发 Timeout`按钮，等待10多秒后，应用会触发 TimeOut Crash，产生如下日志 


```
D/ghost: =============fire finalize=============FinalizerDaemon
I/.watchdogkille: Thread[3,tid=4369,WaitingInMainSignalCatcherLoop,Thread*=0x76e6ece16400,peer=0x149802d0,"Signal Catcher"]: reacting to signal 3
I/.watchdogkille: Wrote stack traces to '[tombstoned]'
E/AndroidRuntime: FATAL EXCEPTION: FinalizerWatchdogDaemon
    Process: com.dodola.watchdogkiller, PID: 4363
    java.util.concurrent.TimeoutException: com.dodola.watchdogkiller.GhostObject.finalize() timed out after 10 seconds
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:373)
        at java.lang.Thread.sleep(Thread.java:314)
        at com.dodola.watchdogkiller.GhostObject.finalize(GhostObject.java:13)
        at java.lang.Daemons$FinalizerDaemon.doFinalize(Daemons.java:250)
        at java.lang.Daemons$FinalizerDaemon.runInternal(Daemons.java:237)
        at java.lang.Daemons$Daemon.run(Daemons.java:103)
        at java.lang.Thread.run(Thread.java:764)
I/Process: Sending signal. PID: 4363 SIG: 9

```
3. 点击`Kill WatchDog` 按钮可以关闭 Timeout watchdog，然后点击`触发 TimeOut` 按钮观察情况，正常情况下不会产生 crash

疑问点
===
如果直接调用Daemons$FinalizerWatchdogDaemon的stop方法，在Android 6.0之前的版本可能会有问题。
```
final Class clazz = Class.forName("java.lang.Daemons$FinalizerWatchdogDaemon");
final Field field = clazz.getDeclaredField("INSTANCE");
field.setAccessible(true);
final Object watchdog = field.get(null);
final Method method = clazz.getSuperclass().getDeclaredMethod("stop");
method.setAccessible(true);
method.invoke(watchdog);
```

这是为什么，你能告诉我们吗？


### 源码分析

Daemons类有好几个静态内部类，都是守护线程
1. FinalizerDaemon和ReferenceQueueDaemon用于垃圾回收
1. FinalizerWatchdogDaemon：看门狗线程，用于监控垃圾回收线程卡死



### 主要逻辑
#### 死循环
每次检查当前处理进度，然后休眠，若休眠结束后还在处理当前对象，则认为超时

### 技巧
休眠分五次
1. 提前检测完成：每次醒来检查是否完成，避免不必要的长时间休眠
1. 避免虚假超时：分段给了根据进程时钟调整剩余休眠时间的机会
    1. System.nanoTime() 进程时钟
    1. Thread.sleep() 墙钟时间，系统时间，可修改

### 解决超时异常逻辑
FinalizerWatchdogDaemon中协作条件是isRunning方法
``` java
        @Override public void runInternal() {
            while (isRunning()) {
                if (!sleepUntilNeeded()) {
                    // We have been interrupted, need to see if this daemon has been stopped.
                    continue;
                }
                final TimeoutException exception = waitForProgress();
                if (exception != null && !VMDebug.isDebuggerConnected()) {
                    timedOut(exception);
                    break;
                }
            }
        }
```
isRunning方法是判断thread变量是否为null
``` java
        @UnsupportedAppUsage
        protected synchronized boolean isRunning() {
            return thread != null;
        }
```
所以只需要反射将thread设为null，即可终止FinalizerWatchdogDaemon线程

``` java
            final Class clazz = Class.forName("java.lang.Daemons$FinalizerWatchdogDaemon");
            final Field field = clazz.getDeclaredField("INSTANCE");
            field.setAccessible(true);
            final Object watchdog = field.get(null);
            try {
                final Field thread = clazz.getSuperclass().getDeclaredField("thread");
                thread.setAccessible(true);
                thread.set(watchdog, null);
            } catch (final Throwable t) {
                Log.e(TAG, "stopWatchDog, set null occur error:" + t);
                }

```
AndroidP之后FinalizerWatchdogDaemon无法反射，会抛异常


### 元反射
高版本可用元反射绕过系统黑名单限制，但不稳定
1. 修改 VMRuntime.setHiddenApiExemptions
1. 修改 Class.getDeclaredField 的访问检查


# 其他
Reference有 FinalizerReference子类，内部也有个ReferenceQueue
需要研究各个守护线程逻辑

Daemons类不是Java平台的，是Android特有的
Java中finalization逻辑完全封装在jvm中，无法干涉