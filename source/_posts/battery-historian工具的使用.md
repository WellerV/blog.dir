---
title: battery-historian工具的使用
date: 2017/11/16 13:44:00
author: 胡玮
tags:
- Android 
categories: 
- android功耗优化
---

本篇主要介绍google官方功耗测量工具battery-historian的使用
<!--more-->
## 一，battery-historian工具的安装
### 1，安装Docker CE
以下都是ubuntu环境，
卸载旧版本：
```
$ sudo apt-get remove docker docker-engine docker.io
```
1) 更新源
```
$ sudo apt-get update
```
2) 安装最新版本的docker-ce
```
 $ sudo apt-get install docker-ce
```
3) 验证docker是否安装成功

x86_64:
```
$ sudo docker run hello-world
```

armhf:
```
$ sudo docker run armhf/hello-world
```
运行示例：
![enter description here][1]

### 2，阿里云docker hub加速
由于国内无法访问到google的docker hub,所以我们需要利用阿里云的镜像来进行加速。
#### 获取独有的加速器地址
关于加速器地址，需要登录[容器Hub服务][2]，左侧的加速器帮助页面就会显示为你独立分配的加速地址。
```
公网Mirror：[系统分配前缀].mirror.aliyuncs.com
```
#### 配置阿里云加速器
```
udo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["<your accelerate address>t
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
替换成上面获取的加速地址

### 3，运行battery-historian
由于阿里云上已经有了对应的google_battery．所以直接运行以下命令即可：
```
$ sudo docker run -p <port>:9999 registry.cn-beijing.aliyuncs.com/center1/google_battery
```
＜port＞替换自己想要设置的本地端口号
运行效果如下：
![enter description here][3]

打开[http://localhost:9999/][4]，
![enter description here][5]

## 二，battery-historian工具的使用
### 1，常见命令
#### 1) 重置电量统计
```
$ adb shell dumpsys batterystats --reset
```
#### 2) 导出bug report
7.0以及以上：
```
$ adb bugreport bugreport.zip
```
6.0以及以下：
```
$ adb bugreport > bugreport.txt
```
#### 3) 详细记录唤醒锁的信息
```
adb shell dumpsys batterystats --enable full-wake-history
```

### 2，参数解释

#### 1) 系统视图
![enter description here][6]

Add Metrics表示可以添加额外的参数，比如Charing status(充电状态)，Health(电池健康).
　
图标的x坐标是时间轴，y坐标是一些参数的状态图，将鼠标移动到图表上，可以看到更具体的信息．中间的黑色折线表示电池的电量状态．
其他参数表示的信息如下所示：
- CPU running(cpu运行的状态)
- App Processor Wakeup(应用程序处理器唤醒)
- Kernel only uptime(内核uptime)
- Userspace wakelock(用户空间wakelock)
- Long wakelocks
- Screen(屏幕耗电)
- Top App(上层应用)
- Activity Manager Proc(AMS耗电)
- Crashes(logcat) (crash后输出日志)
- Doze (Doze模式)
- JobScheduler
- SyncManager
- GPS
- Phone State
- Network connectivity(网络连接)
- Mobile signal strength
- Wifi full lock
- Wifi scan
- Wifi radio(wifi无线电)
- Foreground process (前台进程)
- Temperature(温度)


#### 2) app视图
![enter description here][7]
以＂腾讯视频＂应用为例，app视图分为基本信息，网络信息，wakelocks,services以及进程信息和传感器信息等．
基本信息中包含
- 进程名
- UID
- cpu耗时
- 耗电量百分比

网络信息中包含具体的网络类型以及扫描次数．
services里面显示了具体的services以及启动次数和时间．
进程信息中包含用户操作时间和系统消耗时间以及处于前台的时间，还有启动,ANR以及crashes次数．

  [1]: http://on8vjlgub.bkt.clouddn.com/docker-helloworld.png "docker-helloworld"
  [2]: https://cr.console.aliyun.com/?spm=5176.100239.blogcont29941.12.bBjSmY
  [3]: http://on8vjlgub.bkt.clouddn.com/docker_Battery_Historian.png "docker_Battery_Historian"
  [4]: http://localhost:9999/
  [5]: http://on8vjlgub.bkt.clouddn.com/localhost.png "localhost"
  [6]: http://on8vjlgub.bkt.clouddn.com/battery_chart.png "battery_chart"
  [7]: http://on8vjlgub.bkt.clouddn.com/app-battery.png "app-battery"
