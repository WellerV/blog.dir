---

title: 热更新Tinker研究（三）：加载补丁
date: 2017/03/22 17:41:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---
 

# 热更新Tinker研究（三）：加载补丁
本文主要讲解Tinker加载patch.apk的过程，主要是研究当把patch_signed_7zip.apk推送到sdcard之后，点击LOAD  PATCH按钮之后的流程分析。
```
        loadPatchButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), Environment.getExternalStorageDirectory().getAbsolutePath() + "/patch_signed_7zip.apk");
            }
        });
```
patch的整个过程如下所示：
```flow 
st=>start: 开始
e=>end: 结束
op1=>operation: 接收patchFile
op2=>operation: 应用目录加载patch
op3=>operation: 写入header和map
op4=>operation: tryPatch操作
op5=>operation: 检查odex文件
op6=>operation: 处理结果result

st->op1->op2->op3->op4->op5->op6->e
``` 
类之间的关系图如下：
 ![enter description here][1]

## 一、准备知识
首先我们知道patch.apk是整个更新信息的载体，在加载补丁之前，我们需要弄清楚patch.apk的结构以及各个文件的作用。
### 1，patch.apk的主要结构
.
├── ./assets
│   ├── ./assets/dex_meta.txt	dex相关信息
│   ├── ./assets/only_use_to_test_tinker_resource.txt	测试文件
│   ├── ./assets/package_meta.txt	package相关信息
│   └── ./assets/res_meta.txt	resource变化信息
├── ./classes.dex   更新dex文件(非常规dex格式)
├── ./META-INF	 签名目录
│   ├── ./META-INF/ANDROIDD.RSA
│   ├── ./META-INF/ANDROIDD.SF
│   └── ./META-INF/MANIFEST.MF
├── ./res		变化的res目录
│   ├── ./res/layout
│   │   └── ./res/layout/activity_main.xml
│   └── ./res/layout-v17
│       └── ./res/layout-v17/activity_main.xml
└── ./test.dex	测试dex文件
### 2，package_meta.txt
下面是一个测试的package_meta的例子
```txt
#base package config field
#Thu Mar 16 11:29:04 CST 2017
platform=all
NEW_TINKER_ID=tinker_id_9d2b250
TINKER_ID=tinker_id_9d2b250
patchMessage=tinker is sample to use
patchVersion=1.0
```
除了NEW_TINKER_ID和TINKER_ID,其他的字段主要对应build.gradle中的属性packageConfig自定义的字段值
```groovy
        packageConfig {
            /**
             * optional，default 'TINKER_ID, TINKER_ID_VALUE' 'NEW_TINKER_ID, NEW_TINKER_ID_VALUE'
             * package meta file gen. path is assets/package_meta.txt in patch file
             * you can use securityCheck.getPackageProperties() in your ownPackageCheck method
             * or TinkerLoadResult.getPackageConfigByName
             * we will get the TINKER_ID from the old apk manifest for you automatic,
             * other config files (such as patchMessage below)is not necessary
             */
            configField("patchMessage", "tinker is sample to use")
            /**
             * just a sample case, you can use such as sdkVersion, brand, channel...
             * you can parse it in the SamplePatchListener.
             * Then you can use patch conditional!
             */
            configField("platform", "all")
            /**
             * patch version via packageConfig
             */
            configField("patchVersion", "1.0")
        }
```
自定义一些变量，用于在版本之间传递，可以更多地自己去控制流程，方便信息的扩展。
### 3，dex_meta.txt
下面是一个dex_meta.txt的实例
```
classes.dex,,7e197a86bfc55b2345e49254670d13ad,7e197a86bfc55b2345e49254670d13ad,212f0b11b6ff37a4153c8bb6e572d3ef,1880959952,jar
test.dex,,56900442eb5b7e1de45449d0685e6e00,56900442eb5b7e1de45449d0685e6e00,0,0,jar
```
用表格描述
| name  | path | destMd5InDvm | destMd5InArt | dexDiffMd5 | oldDexCrc | dexMode |
| -------  | ----- | ------------------  | ----------------- | --------------- | ------------- | ------------|
|classes.dex||7e197a86bfc55b2345e49254670d13ad|7e197a86bfc55b2345e49254670d13ad|212f0b11b6ff37a4153c8bb6e572d3ef|1880959952|jar|
|test.dex||56900442eb5b7e1de45449d0685e6e00|56900442eb5b7e1de45449d0685e6e00|0|0|jar|
destMd5InDvm和destMd5InArt分别对应不同虚拟机下生成的dex文件的md5，因为需要合成的dex文件的md5其实已经知道了，所以这里主要用于校验。dexDiffMd5是当前补丁dex对应的md5。dexMode是指dex文件的压缩模式，有jar和dex两种。

### 4，res_meta.txt
```
resources_out.zip,1310764963,7d766f7aeead0c72a6479d55d255c611
pattern:3
resources.arsc
res/*
assets/*
modify:2
res/layout/activity_main.xml
res/layout-v17/activity_main.xml
add:1
assets/only_use_to_test_tinker_resource.txt
```
包含生成资源包文件，md5，以及过滤目录，修改增加信息等等。

## 二、日志分析jieguo
下面的日志截取真实环境，并做了一些删减，结合日志，能够更方便地帮我们分析整个load patch的过程。下面的日志为方便阅读源码使用，不必深究。
### 接收Patch File
TinkerInstaller接收到patch file,并启动TinkerPatchService。
```
03-16 14:41:57.362 8508-8508/tinker.sample.android I/Tinker.SamplePatchListener: receive a patch file: /storage/emulated/0/patch_signed_7zip.apk, file size:7322
03-16 14:41:57.401 8508-8508/tinker.sample.android W/Tinker.UpgradePatchRetry: onPatchListenerCheck retry file is not exist, just return
03-16 14:41:57.408 8508-8508/tinker.sample.android I/Tinker.SamplePatchListener: get platform:all
03-16 14:41:57.412 1169-1426/system_process V/ActivityManager: startService: Intent { cmp=tinker.sample.android/com.tencent.tinker.lib.service.TinkerPatchService (has extras) } callerApp=ProcessRecord{f11eaa9 8508:tinker.sample.android/u0a172}
```

### 应用目录加载patch
从应用程序目录检查patchInfo,并进行补丁加载
```
03-16 14:41:57.777 8958-8958/tinker.sample.android:patch W/Tinker.TinkerLoader: tryLoadPatchFiles:patch dir not exist:/data/user/0/tinker.sample.android/tinker
03-16 14:41:57.782 8958-8958/tinker.sample.android:patch D/Tinker.DefaultAppLike: onBaseContextAttached:
03-16 14:41:57.808 8958-8958/tinker.sample.android:patch I/Tinker.SamplePatchListener: application maxMemory:256
03-16 14:41:57.817 8958-8958/tinker.sample.android:patch W/Tinker.Tinker: tinker patch directory: /data/user/0/tinker.sample.android/tinker
03-16 14:41:57.823 8958-8958/tinker.sample.android:patch I/Tinker.Tinker: try to install tinker, isEnable: true, version: 1.7.7
03-16 14:41:57.826 8958-8958/tinker.sample.android:patch I/Tinker.TinkerLoadResult: parseTinkerResult loadCode:-2, systemOTA:false
03-16 14:41:57.827 8958-8958/tinker.sample.android:patch W/Tinker.TinkerLoadResult: can't find patch file, is ok, just return
03-16 14:41:57.828 8958-8958/tinker.sample.android:patch I/Tinker.DefaultLoadReporter: patch loadReporter onLoadResult: patch load result, path:/data/user/0/tinker.sample.android/tinker, code:-2, cost:6
03-16 14:41:57.829 8958-8958/tinker.sample.android:patch W/Tinker.Tinker: tinker load fail!
03-16 14:41:57.832 8958-8958/tinker.sample.android:pa缺少tch D/Tinker.DefaultAppLike: onCreate
```
### tryPatch
```
03-16 14:41:58.102 8958-8980/tinker.sample.android:patch I/Tinker.UpgradePatch:     public void onPatchRetryLoad() {
        if (!isRetryEnable) {
            TinkerLog.w(TAG, "onPatchRetryLoad retry disabled, just return");
            return;
        }
        Tinker tinker = Tinker.with(context);
        //only retry on main process
        if (!tinker.isMainProcess()) {
            TinkerLog.w(TAG, "onPatchRetryLoad retry is not main process, just return");
            return;
        }

        if (!retryInfoFile.exists()) {
            TinkerLog.w(TAG, "onPatchRetryLoad retry info not exist, just return");
            return;
        }

        if (TinkerServiceInternals.isTinkerPatchServiceRunning(context)) {
            TinkerLog.w(TAG, "onPatchRetryLoad tinker ser缺少vice is running, just return");
            return;
        }
        //must use temp file
        String path = tempPatchFile.getAbsolutePath();
        if (path == null || !new File(path).exists()) {
            TinkerLog.w(TAG, "onPatchRetryLoad patch file: %s is not exist, just return", path);
            return;
        }
        TinkerLog.w(TAG, "onPatchRetryLoad patch file: %s is exist, retry to patch", path);
        TinkerInstaller.onReceiveUpgradePatch(context, path);
        SampleTinkerReport.onReportRetryPatch();
    } tryPatch:patchMd5:9b4fece1c1516b04bdbe7c90b7832910
03-16 14:41:58.102 8958-8980/tinker.sample.android:patch I/Tinker.UpgradePatch: UpgradePatch tryPatch:patchVersionDirectory:/data/user/0/tinker.sample.android/tinker/patch-9b4fece1
03-16 14:41:58.109 8958-8980/tinker.sample.android:patch W/Tinker.UpgradePatch: UpgradePatch copy patch file, src file: /storage/emulated/0/patch_signed_7zip.apk size: 7322, dest file: /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/patch-9b4fece1.apk size:7322
03-16 14:42:00.789 8958-8980/tinker.sample.android:patch W/Tinker.DexDiffPatchInternal: success recover dex file: /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/classes.dex.jar, size: 849611, use time: 2642
03-16 14:42:00.791 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: try Extracting /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/test.dex.jar
03-16 14:42:00.795 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: isExtractionSuccessful: true
03-16 14:42:00.797 8958-8980/tinker.sample.android:patch W/Tinker.DexDiffPatchInternal: patch recover, try to optimize dex file count:2


03-16 14:42:00.804 8958-9022/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: start to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/classes.dex.jar, size: 849611
03-16 14:42:00.804 8958-9023/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: start to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/test.dex.jar, size: 470
                                                                                        
                                                                                        [ 03-16 14:42:00.849  1169: 1214 E/         ]
                                                                                        CWM:[999]readEvents:In
                                                                                        
                                                                                        
                                                                                        [ 03-16 14:42:00.849  1169: 1214 E/         ]
                                                                                        CWM:[1446]processEvent:Id=0,data:0.05:0.02:9.73
                                                                                        
                                                                                        
                                                                                        [ 03-16 14:42:00.849  1169: 1214 E/         ]
                                                                                        CWM:[1059]readEvents:Out:numEventReceived[1]
03-16 14:42:00.947 8958-9023/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: success to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/test.dex.jar, opt file size: 8872, use time 143
03-16 14:42:01.311 8958-9022/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: success to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/classes.dex.jar, opt file size: 2093736, use time 507
03-16 14:42:01.312 8958-8980/tinker.sample.android:patch I/Tinker.ParallelDex: All dexes are optimized successfully, cost: 512 ms.
03-16 14:42:01.314 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: recover dex result:true, cost:3195
03-16 14:42:01.319 8958-8980/tinker.sample.android:patch W/Tinker.BsDiffPatchInternal: patch recover, library is not contained
                                                                                       
                                                                                       [ 03-16 14:42:01.326  1169: 1214 E/         ]
                                                                                       CWM:[999]readEvents:In
                                                                                       
                                                                                       
                                                                                       [ 03-16 14:42:01.326  1169: 1214 E/         ]
                                                                                       CWM:[1446]processEvent:Id=0,data:0.05:0.01:9.74
                                                                                       
                                                                                       
                                                                                       [ 03-16 14:42:01.326  1169: 1214 E/         ]
                                                                                   [缺少类关系图]    CWM:[1059]readEvents:Out:numEventReceived[1]
03-16 14:42:01.328 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: res dir: /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/res/, meta: resArscMd5:7d766f7aeead0c72a6479d55d255c611
                                                                                        arscBaseCrc:1310764963
                                                                                        pattern:res/.*
                                                                                        pattern:assets/.*
                                                                                        pattern:resources\.arsc
                                                                                        addedSet:assets/only_use_to_test_tinker_resource.txt
                                                                                        modifiedSet:res/layout/activity_main.xml
                                                                                        modifiedSet:res/layout-v17/activity_main.xml
03-16 14:42:01.346 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: no large modify resources, just return
03-16 14:42:01.357 8958-8980/tinker.sample.android:patch W/art: Method processed more than once: void com.tencent.tinker.commons.ziputil.TinkerZipEntry.<init>(byte[], java.io.InputStream, java.nio.charset.Charset, boolean)
03-16 14:42:01.575 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: final new resource file:/data/user/0/tinker.sample.android/tinker/patch-9b4fece1/res/resources.apk, entry count:327, size:438132
03-16 14:42:01.575 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: recover resource result:true, cost:251
03-16 14:42:01.576 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: dex count: 2, final wait time: 1203-16 14:41:58.102 8958-8980/tinker.sample.android:patch I/Tinker.UpgradePatch: UpgradePatch tryPatch:patchMd5:9b4fece1c1516b04bdbe7c90b7832910
03-16 14:41:58.102 8958-8980/tinker.sample.android:patch I/Tinker.UpgradePatch: UpgradePatch tryPatch:patchVersionDirectory:/data/user/0/tinker.sample.android/tinker/patch-9b4fece1
03-16 14:41:58.109 8958-8980/tinker.sample.android:patch W/Tinker.UpgradePatch: UpgradePatch copy patch file, src file: /storage/emulated/0/patch_signed_7zip.apk size: 7322, dest file: /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/patch-9b4fece1.apk size:7322
03-16 14:42:00.789 8958-8980/tinker.sample.android:patch W/Tinker.DexDiffPatchInternal: success recover dex file: /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/classes.dex.jar, size: 849611, use time: 2642
03-16 14:42:00.791 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: try Extracting /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/test.dex.jar
03-16 14:42:00.795 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: isExtractionSuccessful: true
03-16 14:42:00.797 8958-8980/tinker.sample.android:patch W/Tinker.DexDiffPatchInternal: patch recover, try to optimize dex file count:2


03-16 14:42:00.804 8958-9022/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: start to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/classes.dex.jar, size: 849611
03-16 14:42:00.804 8958-9023/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: start to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/test.dex.jar, size: 470
                                                                                        
                                                                                        [ 03-16 14:42:00.849  1169: 1214 E/         ]
                                                                                        CWM:[999]readEvents:In
                                                                                        
                                                                                        
                                                                                        [ 03-16 14:42:00.849  1169: 1214 E/         ]
                                                                                        CWM:[1446]processEvent:Id=0,data:0.05:0.02:9.73
                                                                                        
                                                                                        
                                                                                        [ 03-16 14:42:00.849  1169: 1214 E/         ]
                                                                                        CWM:[1059]readEvents:Out:numEventReceived[1]
03-16 14:42:00.947 8958-9023/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: success to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/test.dex.jar, opt file size: 8872, use time 143
03-16 14:42:01.311 8958-9022/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: success to parallel optimize dex /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/dex/classes.dex.jar, opt file size: 2093736, use time 507
03-16 14:42:01.312 8958-8980/tinker.sample.android:patch I/Tinker.ParallelDex: All dexes are optimized successfully, cost: 512 ms.
03-16 14:42:01.314 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: recover dex result:true, cost:3195
03-16 14:42:01.319 8958-8980/tinker.sample.android:patch W/Tinker.BsDiffPatchInternal: patch recover, library is not contained
                                                                                       
                                                                                       [ 03-16 14:42:01.326  1169: 1214 E/         ]
                                                                                       CWM:[999]readEvents:In
                                                                                       
                                                                                       
                                                                                       [ 03-16 14:42:01.326  1169: 1214 E/         ]
                                                                                       CWM:[1446]processEvent:Id=0,data:0.05:0.01:9.74
                                                                                       
                                                                                       
                                                                                       [ 03-16 14:42:01.326  1169: 1214 E/         ]
                                                                                       CWM:[1059]readEvents:Out:numEventReceived[1]
03-16 14:42:01.328 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: res dir: /data/user/0/tinker.sample.android/tinker/patch-9b4fece1/res/, meta: resArscMd5:7d766f7aeead0c72a6479d55d255c611
                                                                                        arscBaseCrc:1310764963
                                                                                        pattern:res/.*
                                                                                        pattern:assets/.*
                                                                                        pattern:resources\.arsc
                                                                                        addedSet:assets/only_use_to_test_tinker_resource.txt
                                                                                        modifiedSet:res/layout/activity_main.xml
                                                                                        modifiedSet:res/layout-v17/activity_main.xml
03-16 14:42:01.346 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: no large modify resources, just return
03-16 14:42:01.357 8958-8980/tinker.sample.android:patch W/art: Method processed more than once: void com.tencent.tinker.commons.ziputil.TinkerZipEntry.<init>(byte[], java.io.InputStream, java.nio.charset.Charset, boolean)
03-16 14:42:01.575 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: final new resource file:/data/user/0/tinker.sample.android/tinker/patch-9b4fece1/res/resources.apk, entry count:327, size:438132
03-16 14:42:01.575 8958-8980/tinker.sample.android:patch I/Tinker.ResDiffPatchInternal: recover resource result:true, cost:251
03-16 14:42:01.576 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: dex count: 2, final wait time: 12
```
### 检查odex文件
负责等待DexOpt优化完dex，并对odex文件进行校验。
```
03-16 14:42:01.576 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: dex count: 2, final wait time: 12
03-16 14:42:01.581 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: check dex optimizer file classes.dex.dex, size 2093736
03-16 14:42:01.582 8958-8980/tinker.sample.android:patch I/Tinker.DexDiffPatchInternal: check dex optimizer file test.dex.dex, size 8872
03-16 14:42:01.591 8958-8980/tinker.sample.android:patch W/Tinker.UpgradePatch: UpgradePatch tryPatch: done, it is ok
```
### 处理Result
针对patch返回的结果进行一些相应的操作
```
03-16 14:42:01.592 8958-8980/tinker.sample.android:patch I/Tinker.PatchFileUtil: safeDeleteFile, try to delete path: /data/user/0/tinker.sample.android/tinker_temp/temp.apk
03-16 14:42:01.680 8508-9051/tinker.sample.android I/Tinker.SampleResultService: SampleResultService receive result: 
                                                                                 PatchResult: 
                                                                                 isSuccess:true
                                                                                 rawPatchFilePath:/storage/emulated/0/patch_signed_7zip.apk
                                                                                 costTime:3638
                                                                                 patchVersion:9b4fece1c1516b04bdbe7c90b7832910
03-16 14:42:01.704 8508-9051/tinker.sample.android W/Tinker.DefaultTinkerResultService: deleteRawPatchFile rawFile path: /storage/emulated/0/patch_signed_7zip.apk
03-16 14:42:01.704 8508-9051/tinker.sample.android I/Tinker.PatchFileUtil: safeDeleteFile, try to delete path: /storage/emulated/0/patch_signed_7zip.apk
03-16 14:42:01.706 8508-9051/tinker.sample.android I/Tinker.SampleResultService: tinker wait screen to restart process
```

## 三、接收PatchFile
TinkerInstaller的onReceiveUpgradePatch()最终会调用SamplePatchListener的onPatchReceived（）方法。

### patchCheck
主要是进行patch listener状态的检测，包含tinker开关、文件是否合法、进程状态、Service状态、平台属性的检测。
```java
    protected int patchCheck(String path) {
        Tinker manager = Tinker.with(context);
        //check SharePreferences also
        if (!manager.isTinkerEnabled() || !ShareTinkerInternals.isTinkerEnableWithSharedPreferences(context)) {
            return ShareConstants.ERROR_PATCH_DISABLE;
        }
        File file = new File(path);

        if (!SharePatchFileUtil.isLegalFile(file)) {
            return ShareConstants.ERROR_PATCH_NOTEXIST;
        }

        //patch service can not send request
        if (manager.isPatchProcess()) {
            return ShareConstants.ERROR_PATCH_INSERVICE;
        }

        //if the patch service is running, pending
        if (TinkerServiceInternals.isTinkerPatchServiceRunning(context)) {
            return ShareConstants.ERROR_PATCH_RUNNING;
        }
        return ShareConstants.ERROR_PATCH_OK;
    }

```
### 启动TinkerPatchService
TinkerPatchService运行在单独的进程xxx:process里面,由于是IntentService,也是在单独的线程里面运行，并且会启动后一个InnerSerivce来提高优先级。

真正执行patch的对象是upgradePatchProcessor，由resultServiceClass来接收结果。
```java
public class TinkerPatchService extends IntentService {
	 ...
    private static       AbstractPatch upgradePatchProcessor = null;
 
    private static       Class<? extends AbstractResultService> resultServiceClass   = null;
    ...
    }
```
然后为了提高Service的优先级，除了startForeground（），还会启动一个InnerService。

## 四、应用目录加载patch
application的onBaseAttach()每次在context重新创建时会调用，主要是去应用目录下加载patch.apk，核心代码如下：
```java
    public void onPatchRetryLoad() {
    //isRetryEnable开关
        if (!isRetryEnable) {
            TinkerLog.w(TAG, "onPatchRetryLoad retry disabled, just return");
            return;
        }
        Tinker tinker = Tinker.with(context);
        //only retry on main process
        if (!tinker.isMainProcess()) {
            TinkerLog.w(TAG, "onPatchRetryLoad retry is not main process, just return");
            return;
        }

        if (!retryInfoFile.exists()) {
            TinkerLog.w(TAG, "onPatchRetryLoad retry info not exist, just return");
            return;
        }

        if (TinkerServiceInternals.isTinkerPatchServiceRunning(context)) {
            TinkerLog.w(TAG, "onPatchRetryLoad tinker service is running, just return");
            return;
        }
        //去取temp.apk
        String path = tempPatchFile.getAbsolutePath();
        if (path == null || !new File(path).exists()) {
            TinkerLog.w(TAG, "onPatchRetryLoad patch file: %s is not exist, just return", path);
            return;
        }
        TinkerLog.w(TAG, "onPatchRetryLoad patch file: %s is exist, retry to patch", path);
        //检查temp路径下的patch文件情况
        TinkerInstaller.onReceiveUpgradePatch(context, path);
        SampleTinkerReport.onReportRetryPatch();
    }
```
## 五、tryPatch
### 校验
#### 文件合法性校验
```java
        if (!SharePatchFileUtil.isLegalFile(patchFile)) {
            TinkerLog.e(TAG, "UpgradePatch tryPatch:patch file is not found, just return");
            return false;
        }
```
#### signature校验
首先通过包管理器获取publicKey
```java
            PackageManager pm = context.getPackageManager();
            String packageName = context.getPackageName();
            PackageInfo packageInfo = pm.getPackageInfo(packageName, PackageManager.GET_SIGNATURES);
            CertificateFactory certFactory = CertificateFactory.getInstance("X.509");
            stream = new ByteArrayInputStream(packageInfo.signatures[0].toByteArray());
            X509Certificate cert = (X509Certificate) certFactory.generateCertificate(stream);
            mPublicKey = cert.getPublicKey();
```
然后去校验apk中文件的签名，为了加快效率，这里只检查meta.txt结尾的几个文件。主要是dex_meta.txt，res_mete.txt等等。
```java
 public boolean verifyPatchMetaSignature(File path) {
        if (!SharePatchFileUtil.isLegalFile(path)) {
            return false;
        }
        JarFile jarFile = null;
        try {
            jarFile = new JarFile(path);
            final Enumeration<JarEntry> entries = jarFile.entries();
            while (entries.hasMoreElements()) {
                JarEntry jarEntry = entries.nextElement();
                // no code
                if (jarEntry == null) {
                    continue;
                }

                final String namwenje = jarEntry.getName();
                if (name.startsWith("META-INF/")) {
                    continue;
                }
                //for faster, only check the meta.txt files
                //we will check other files's mad5 written in meta files
                if (!name.endsWith(ShareConstants.META_SUFFIX)) {
                    continue;
                }
                metaContentMap.put(name, SharePatchFileUtil.loadDigestes(jarFile, jarEntry));
                Certificate[] certs = jarEntry.getCertificates();
                if (certs == null) {
                    return false;
                }
                if (!check(path, certs)) {
                    return false;
                }
            }
        } catch (Exception e) {
            throw new TinkerRuntimeException(
                String.format("ShareSecurityCheck file %s, size %d verifyPatchMetaSignature fail", path.getAbsolutePath(), path.length()), e);
        } finally {
            try {
                if (jarFile != null) {
                    jarFile.close();
                }
            } catch (IOException e) {
                Log.e(TAG, path.getAbsolutePath(), e);
            }
        }
        return true;
    }

```
#### tinkerId校验
baseApk的tinkerId是会声明在AndroidManifest.xml中，
![enter description here][3]
而patch.apk则是存在于/assets/package_meta.txt中
![enter description here][4]
此过程主要为了检查两个tinker_id是否一致
```
		//从mainifest或区域oldTinkerId
        String oldTinkerId = getManifestTinkerID(context);
        if (oldTinkerId == null) {
            return ShareConstants.ERROR_PACKAGE_CHECK_APK_TINKER_ID_NOT_FOUND;
        }

        HashMap<String, String> properties = securityCheck.getPackagePropertiesIfPresent();

        if (properties == null) {
            return ShareConstants.ERROR_PACKAGE_CHECK_PACKAGE_META_NOT_FOUND;
        }
		//获取patchTinkerId并进行对比
        String patchTinkerId = properties.get(ShareConstants.TINKER_ID);
        if (patchTinkerId == null) {
            return ShareConstants.ERROR_PACKAGE_CHECK_PATCH_TINKER_ID_NOT_FOUND;
        }
        if (!oldTinkerId.equals(patchTinkerId)) {
            Log.e(TAG, "tinkerId is not equal, base is " + oldTinkerId + ", but patch is " + patchTinkerId);
            return ShareConstants.ERROR_PACKAGE_CHECK_TINKER_ID_NOT_EQUAL;
        }
        return ShareConstants.ERROR_PACKAGE_CHECK_OK;
```
#### tinkerFlag标记
根据不同的flag去开启或者关闭对应的功能
有如下一些标记
```
	//关闭tinker
    public static final int TINKER_DISABLE             = 0x00;  
    //开启dex
    public static final int TINKER_DEX_MASK            = 0x01;
    //开启nativeLarary
    public static final int TINKER_NATIVE_LIBRARY_MASK = 0x02;
    //开启resource
    public static final int TINKER_RESOURCE_MASK       = 0x04;
    //同时开启dex和resource
    public static final int TINKER_DEX_AND_LIBRARY     = TINKER_DEX_MASK | TINKER_NATIVE_LIBRARY_MASK;
    //开启所有功能
    public static final int TINKER_ENABLE_ALL          = TINKER_DEX_MASK | TINKER_NATIVE_LIBRARY_MASK | TINKER_RESOURCE_MASK;
```
进行功能设置
```
        //check dex
        boolean dexEnable = isTinkerEnabledForDex(tinkerFlag);
        if (!dexEnable && metaContentMap.containsKey(ShareConstants.DEX_META_FILE)) {
            return ShareConstants.ERROR_PACKAGE_CHECK_TINKERFLAG_NOT_SUPPORT;
        }
        //check native library
        boolean nativeEnable = isTinkerEnabledForNativeLib(tinkerFlag);
        if (!nativeEnable && metaContentMap.containsKey(ShareConstants.SO_META_FILE)) {
            return ShareConstants.ERROR_PACKAGE_CHECK_TINKERFLAG_NOT_SUPPORT;
        }
        //check resource
        boolean resEnable = isTinkerEnabledForResource(tinkerFlag);
        if (!resEnable && metaContentMap.containsKey(ShareConstants.RES_META_FILE)) {
            return ShareConstants.ERROR_PACKAGE_CHECK_TINKERFLAG_NOT_SUPPORT;
        }
```
### 对比SharePatchInfo
```
public class SharePatchInfo {
	...
    public String oldVersion;
    public String newVersion;
    public String fingerPrint;
    ...
}
```
oldVersion是patch之前的版本，newVersion是指升级后的版本，
fingerPrint由下面这些参数提供:
```
    private static String deriveFingerprint() {
        String finger = SystemProperties.get("ro.build.fingerprint");
        if (TextUtils.isEmpty(finger)) {
            finger = getString("ro.product.brand") + '/' +
                    getString("ro.product.name") + '/' +
                    getString("ro.product.device") + ':' +
                    getString("ro.build.version.release") + '/' +
                    getString("ro.build.id") + '/' +
                    getString("ro.build.version.incremental") + ':' +
                    getString("ro.build.type") + '/' +
                    getString("ro.build.tags");
        }
        return finger;
    }
```
### tryRecoverDexFiles
### tryRecoverLibraryFiles
### tryRecoverResourceFiles

## 六、检查odex文件
waitDexOptFile
由于部分手机dex2oat过程是异步的，所以需要进行检测dex2oat是否确实生成，
```
 protected static boolean waitDexOptFile() {
        if (optFiles.isEmpty()) {
            return true;
        }

        int size = optFiles.size() * 6;
        if (size > MAX_WAIT_COUNT) {
            size = MAX_WAIT_COUNT;
        }
        TinkerLog.i(TAG, "dex count: %d, final wait time: %d", optFiles.size(), size);

		//检查size的次数，每次检查休眠WAIT_ASYN_OAT_TIME时间，这里是8s。
        for (int i = 0; i < size; i++) {
            if (!checkAllDexOptFile(optFiles, i + 1)) {
                try {
                    Thread.sleep(WAIT_ASYN_OAT_TIME);
                } catch (InterruptedException e) {
                    TinkerLog.e(TAG, "thread sleep InterruptedException e:" + e);
                }
            }
        }

		//最后检查一次，如果还是没有找到，就return
        for (File file : optFiles) {
            TinkerLog.i(TAG, "check dex optimizer file %s, size %d", file.getName(), file.length());

            if (!SharePatchFileUtil.isLegalFile(file)) {
                TinkerLog.e(TAG, "final parallel dex optimizer file %s is not exist, return false", file.getName());
                // don't report fail
//                manager.getPatchReporter()
//                    .onPatchDexOptFail(patchFile, file, file.getParentFile().getPath(),
//                        file.getName(), new TinkerRuntimeException("dexOpt file:" + file.getName() + " is not exist"));
                return false;

            }
        }
        return true;
    }
```
## 七、处理result
DefaultTinkerResultService处理相关逻辑,主要是onPatchResult()的回调，
```
    public void onPatchResult(PatchResult result) {
        if(result == null) {
            TinkerLog.e("Tinker.DefaultTinkerResultService", "DefaultTinkerResultService received null result!!!!", new Object[0]);
        } else {
            TinkerLog.i("Tinker.DefaultTinkerResultService", "DefaultTinkerResultService received a result:%s ", new Object[]{result.toString()});
            TinkerServiceInternals.killTinkerPatchServiceProcess(this.getApplicationContext());
            if(result.isSuccess) {
                this.deleteRawPatchFile(new File(result.rawPatchFilePath));
                if(this.checkIfNeedKill(result)) {
                    Process.killProcess(Process.myPid());
                } else {
                    TinkerLog.i("Tinker.DefaultTinkerResultService", "I have already install the newly patch version!", new Object[0]);
                }
            }

        }
    }
```


  [1]: http://on8vjlgub.bkt.clouddn.com/patchFlow.png "patchFlow"
  [3]: http://on8vjlgub.bkt.clouddn.com/tinker_id.png "tinker_id"
  [4]: http://on8vjlgub.bkt.clouddn.com/meta_tinker_id.png "meta_tinker_id"