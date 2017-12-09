---
title: android电量优化措施 
date: 2017/11/20 13:44:00
author: 胡玮
tags:
- Android 
categories: 
- android功耗优化
---

在开始电量优化以前，我们先总结下设备耗电的一些因素，然后各个击破。
<!--more-->
如下图：
![enter description here][1]
大概包含以下一些因素：
- 屏幕亮度
- 网络相关
- 唤醒，格式模式的切换以及wakelock
- 定位
- 其他传感器

## 一，功耗分析工具
功耗分析的工具多种多样，比如google官方提供的[battery-history][2],腾讯的[GT (Great Tit)][3]。

## 二，针对具体场景进行优化
### 1，保持屏幕
在一段时间后，android设备的屏幕会变暗，直至关闭，然后会停止cpu运行，减少设备的功耗。

但某些场景我们需要来保持屏幕常量，比如电子书阅读器，视频软件，视频聊天软件等等。最好的方式是在Activity中使用FLAG_KEEP_SCREEN_ON 的Flag。
```java
getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
```
只有在特定的界面才会保持屏幕，不像唤醒锁（wake locks),需要申请权限。并且不用担心界面切换以及资源释放问题。

还有一种可以把屏幕唤醒设置到布局上面。
```xml
<LinearLayout
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   android:orientation="vertical"
   android:keepScreenOn="true">
</LinearLayout>
```

我们需要根据具体业务来进行选择

### 2，网络相关
关于网络优化首先我们要了解无线电状态机。
一个完全活跃的无线电设备将耗费大量电量，所以无线电状态机会在各种模式下切换，有以下３种模式：
- 1，Full power：连接处于活动状态时使用，允许设备以最高可能的速率传输数据。
- 2，Low power：在满状态下使用大约50％电池电量的中间状态。
- 3，Standby：没有网络连接处于活动状态或需要的最小能量状态。
虽然低电量和待机状态的电量消耗显着减少，但也会对网络请求造成严重的延迟。
从低状态恢复到全功率需要大约1.5秒，从待机状态转换到满状态可能需要2秒钟。
为了最小化延迟，状态机使用延迟来推迟向较低能量状态的转换。

![enter description here][4]

如果每个时刻都在进行一些零散的网络请求，无线电状态机大部分时间都处于Full power模式。如下图中的Unbundled Transfers：

![enter description here][5]
相反，如果网络请求的时间点相对集中，大部分时间无线电状态机会处于Standby。如图中Bundled Transfers。

具体的网络优化以及测试方法可以看 android网络耗电优化。

### 3，WakeLock
PowerManager 用来控制设备的电源状态. 而PowerManager.WakeLock也称作唤醒锁, 是一种保持 CPU 运转防止设备休眠的方式。例如播放音乐，即使在屏幕关闭时也需要程序在后台运行。

四种唤醒锁：

|   Flag Value            | CPU | 屏幕 | 键盘 |
| ------------------------| --- | ------------ | ------- |
| PARTIAL_WAKE_LOCK       | On* | Off    | Off     |
| SCREEN_DIM_WAKE_LOCK    | On  | Dim逐渐变暗 | Off |
| SCREEN_BRIGHT_WAKE_LOCK | On  | Bright保持亮度 | Off |
| FULL_WAKE_LOCK	  | On  | Bright保持亮度 | Bright保持亮度 |

请注意, 如果是 PARTIAL_WAKE_LOCK, 无论屏幕的状态甚至是用户按了电源钮, CPU 都会继续工作. 如果是其它的唤醒锁, 设备会在用户按下电源钮后停止工作进入休眠状态。

除了上面四种唤醒锁, 还有两种只关乎屏幕显示方式的 flags。

| Flag Value |	描述 |
|---------- | ----- |
| ACQUIRE_CAUSES_WAKEUP |	一旦获得唤醒锁锁时，屏幕和键盘会立即强制打开 |
| ON_AFTER_RELEASE | 释放唤醒锁时 activity timer 会被重置, 屏幕将比平时亮的久一点 |
使用wakelock如下所示：
```java
PowerManager powerManager = (PowerManager) getSystemService(POWER_SERVICE);

// 创建唤醒锁
WakeLock wakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "MyWakelockTag");

// 获得唤醒锁
wakeLock.acquire();

// 进行一些后台服务
....

// 释放唤醒锁, 如果没有其它唤醒锁存在, 设备会很快进入休眠状态
wakelock.release();
```
这里要尽量使用 acquire(long timeout) 设置超时, (也被称作超时锁). 例如网络请求的数据返回时间不确定, 导致本来只需要10s的事情一直等待了1个小时, 这样会使得电量白白浪费了。 设置超时之后, 会自动释放已节省点远。

### 4，定位
选择合适的Location Provider
Android系统支持多个Location Provider：
**GPS_PROVIDER:**
GPS定位，利用GPS芯片通过卫星获得自己的位置信息。定位精准度高，一般在10米左右，耗电量大；但是在室内，GPS定位基本没用。
**NETWORK_PROVIDER：**
网络定位，利用手机基站和WIFI节点的地址来大致定位位置，这种定位方式取决于服务器，即取决于将基站或WIF节点信息翻译成位置信息的服务器的能力。
**PASSIVE_PROVIDER:**
被动定位，就是用现成的，当其他应用使用定位更新了定位信息，系统会保存下来，该应用接收到消息后直接读取就可以了。

#### 及时注销监听
当不需要使用定位时，需要及时注销定位的监听。

#### 多模块使用定位尽量复用
多个模块需要定位时，尽量服用定位信息，比如可以在启动时定位一次，其他定位需要定位时直接去取。
利用PASSIVE_PROVIDER获取其他应用程序得到的定位信息。

## 三，一些补充知识
### 1，Doze模式和App Standby
从 Android 6.0（API 级别 23）开始，Android 引入了两个省电功能，可通过管理应用在设备未连接至电源时的行为方式为用户延长电池寿命。低电耗模式通过在设备长时间处于闲置状态时推迟应用的后台 CPU 和网络 Activity 来减少电池消耗。应用待机模式可推迟用户近期未与之交互的应用的后台网络 Activity。
#### 低电耗模式doze
如果用户设备未插接电源、处于静止状态一段时间且屏幕关闭，设备会进入低电耗模式。 在低电耗模式下，系统会尝试通过限制应用对网络和 CPU 密集型服务的访问来节省电量。 这还可以阻止应用访问网络并推迟其作业、同步和标准闹铃。

系统会定期退出低电耗模式一会儿，好让应用完成其已推迟的 Activity。在此维护时段内，系统会运行所有待定同步、作业和闹铃并允许应用访问网络。

![enter description here][6]

#### 低电耗模式限制
在低电耗模式下，您的应用会受到以下限制：
- 暂停访问网络。
- 系统将忽略 wake locks。
- 标准 AlarmManager 闹铃（包括 setExact() 和 setWindow()）推迟到下一维护时段。
-- 如果您需要设置在低电耗模式下触发的闹铃，请使用 setAndAllowWhileIdle() 或 setExactAndAllowWhileIdle()。
-- 一般情况下，使用 setAlarmClock() 设置的闹铃将继续触发 — 但系统会在这些闹铃触发之前不久退出低电耗模式。
- 系统不执行 Wi-Fi 扫描。
- 系统不允许运行同步适配器。
- 系统不允许运行 JobScheduler。

#### App Standby
应用待机模式允许系统判定应用在用户未主动使用它时处于空闲状态。 当用户有一段时间未触摸应用时，系统便会作出此判定，以下条件均不适用：
- 用户显式启动应用。
- 应用当前有一个进程位于前台（表现为 Activity 或前台服务形式，或被另一 Activity 或前台服务占用）。
- 应用生成用户可在锁屏或通知托盘中看到的通知。

### 2，JobScheduler
JobScheduler和Android 6.0出现的Doze都一样，总结来说就是限制应用频繁唤醒硬件，将任务集中处理，从而达到省电的效果。
当你需要在Android设备满足某种场合才需要去执行处理数据，例如：
- 应用具有您可以推迟的非面向用户的工作（定期数据库数据更新）
- 应用具有当插入设备时您希望优先执行的工作（充电时才希望执行的工作备份数据)
- 需要访问网络或 Wi-Fi 连接的任务(如向服务器拉取内置数据)
- 希望作为一个批次定期运行的许多任务

具体demo可以看　[JobScheduler Sample][7]


  [1]: http://on8vjlgub.bkt.clouddn.com/battry-history-chart.png "battry-history-chart"
  [2]: https://github.com/google/battery-historian
  [3]: https://github.com/Tencent/GT
  [4]: http://on8vjlgub.bkt.clouddn.com/mobile_radio_state_machine.png "无线电状态机"
  [5]: http://on8vjlgub.bkt.clouddn.com/graphs.png "graphs"
  [6]: http://on8vjlgub.bkt.clouddn.com/doze.png "doze"
  [7]: https://github.com/googlesamples/android-JobScheduler/
