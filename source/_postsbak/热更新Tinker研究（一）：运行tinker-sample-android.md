title: 热更新Tinker研究（一）：运行tinker-sample-android
date: 2017/03/15 09:34:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

## 一、导入项目
运行sample项目只需要导入tinker-sample-android工程即可，

![enter description here][1]

整个项目的操作流程分为一下几个步骤：
```flow
st=>start: 开始
e=>end: 结束 
op1=>operation: 编译baseApk(基准apk包)
op2=>operation: 修改源码，生成patchApk
op3=>operation: 推送补丁
op4=>operation: 加载补丁 
st->op1->op2->op3->op4->e 
```

## 二、编译baskApk
### 1，生成baskApk
以下都采用release方式打包，运行assembleRelease
 
 ![enter description here][2]
 
或者采用命令行的方式
```
	./gradlew assembleRelease
```
输出结果:
除了在app/build/outputs/apk下生成app-release.apk外，还会在bak文件下生成相关备份

 ![enter description here][3]

这里默认根据日期和时间生成apk

然后将原始apk安装到设备上

![enter description here][4]

运行:

![enter description here][5]

### 2，指定OldApk相关属性
打开app/build.gradle

![enter description here][6]

这里的tinkerOldApkPath、tinkerApplyMappingPath、tinkerApplyResourcePath三个属性需要和bak文件下baseApk的一致

## 三、修改源码，生成patchApk
### 1, 对工程源码进行修改
这里我只对MainActivity和activity_main.xml进行修改

在MainActivity的onCreate()加入
``` java
   Toast.makeText(this, "I am Patched", Toast.LENGTH_LONG).show();
```
activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                xmlns:tools="http://schemas.android.com/tools"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:paddingBottom="@dimen/activity_vertical_margin"
                android:paddingLeft="@dimen/activity_horizontal_margin"
                android:paddingRight="@dimen/activity_horizontal_margin"
                android:paddingTop="@dimen/activity_vertical_margin"
                tools:context=".app.MainActivity">

    <TextView
        android:id="@+id/textView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Sample patch!"/>

    <Button
        android:id="@+id/loadPatch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_below="@+id/textView"
        android:text="load patch"/>

    <Button
        android:id="@+id/loadLibrary"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_below="@+id/loadPatch"
        android:text="load library"/>

    <Button
        android:id="@+id/cleanPatch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_below="@+id/loadLibrary"
        android:text="clean patch"/>
    <Button
        android:id="@+id/killSelf"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_below="@+id/cleanPatch"
        android:text="kill self"/>
    <Button
        android:id="@+id/showInfo"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_below="@+id/killSelf"
        android:text="show info"/>

	<!-- add by patch-->
    <TextView
        android:layout_below="@id/showInfo"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="I am a patch app"/>
</RelativeLayout>
```
### 2，进行打包并生成patch.apk
运行任务tinkerPatchRelease

![enter description here][7]

得到新的apk和patch.apk文件
 
 ![enter description here][8]
分别输出在app/build/bakApk和 app/build/outputs/tinkerPatch/release/patch_signed_7zip.apk下

## 四、推送补丁
运行
```shell
adb push patch_signed_7zip.apk /storage/sdcard0/
```
将patch_signed_7zip.apk推送到内部存储的根目录

## 五、加载补丁
点击之前运行程序的LOAD PATCH按钮，然后点击KILL PATCH，重新运行程序，发现已经变成了我们修改之后的样子。
 
 ![enter description here][9]
可以看到有了我们添加的toast提示和I am a patch app布局，patch已经生效。

## 六、相关说明
### 1，*-mapping.txt
截取app-release-0314-10-10-21-mapping.txt片段
```log
android.support.annotation.Keep -> android.support.annotation.Keep:
android.support.multidex.MultiDex -> android.support.multidex.a:
    java.lang.String SECONDARY_FOLDER_NAME -> a
    java.util.Set installedApk -> b
    boolean IS_VM_MULTIDEX_CAPABLE -> c
    92:181:void install(android.content.Context) -> a
    188:205:android.content.pm.ApplicationInfo getApplicationInfo(android.content.Context) -> b
    215:234:boolean isVMMultidexCapable(java.lang.String) -> a
    240:249:void installSecondaryDexes(java.lang.ClassLoader,java.io.File,java.util.List) -> a
    256:261:boolean checkValidZipFiles(java.util.List) -> a
    273:288:java.lang.reflect.Field findField(java.lang.Object,java.lang.String) -> b
    302:317:java.lang.reflect.Method findMethod(java.lang.Object,java.lang.String,java.lang.Class[]) -> b
    331:338:void expandFieldArray(java.lang.Object,java.lang.String,java.lang.Object[]) -> b
    341:364:void clearOldDexDir(android.content.Context) -> c
    57:57:java.lang.reflect.Field access$300(java.lang.Object,java.lang.String) -> a
    57:57:void access$400(java.lang.Object,java.lang.String,java.lang.Object[]) -> a
    57:57:java.lang.reflect.Method access$500(java.lang.Object,java.lang.String,java.lang.Class[]) -> a
    63:76:void <clinit>() -> <clinit>
android.support.multidex.MultiDex$V14 -> android.support.multidex.a$a:
    445:449:void install(java.lang.ClassLoader,java.util.List,java.io.File) -> b
    459:462:java.lang.Object[] makeDexElements(java.lang.Object,java.util.ArrayList,java.io.File) -> a
    434:434:void access$100(java.lang.ClassLoader,java.util.List,java.io.File) -> a
```
文件中存放的是类混淆前后的对应关系，比如android.support.multidex.MultiDex -> android.support.multidex.a表示MultiDex会被混淆成 android.support.multidex.a.class。通过反编译app-release-0314-10-10-21.apk进行验证，

![enter description here][10]

### 2, *-R.txt
```
int anim abc_fade_in 0x7f050000
int anim abc_fade_out 0x7f050001
int anim abc_grow_fade_in_from_bottom 0x7f050002
int anim abc_popup_enter 0x7f050003
int anim abc_popup_exit 0x7f050004
int anim abc_shrink_fade_out_from_bottom 0x7f050005
int anim abc_slide_in_bottom 0x7f050006
int anim abc_slide_in_top 0x7f050007
int anim abc_slide_out_bottom 0x7f050008
int anim abc_slide_out_top 0x7f050009
int attr actionBarDivider 0x7f010063
int attr actionBarItemBackground 0x7f010064
int attr actionBarPopupTheme 0x7f01005d

```
R文件的记录

### 3，app/build.gradle
需要用到patch插件
```groovy
apply plugin: 'com.tencent.tinker.patch'
```

以下转载于http://www.jianshu.com/p/d50817b6d622
```groovy
tinkerPatch {
        // 基准apk包的路径，必须输入，否则会报错。
        oldApk = getOldApkPath()
        /**
         * 如果出现以下的情况，并且ignoreWarning为false，我们将中断编译。
         * 因为这些情况可能会导致编译出来的patch包带来风险：
         * case 1: minSdkVersion小于14，但是dexMode的值为"raw";
         * case 2: 新编译的安装包出现新增的四大组件(Activity, BroadcastReceiver...)；
         * case 3: 定义在dex.loader用于加载补丁的类不在main dex中;
         * case 4:  定义在dex.loader用于加载补丁的类出现修改；
         * case 5: resources.arsc改变，但没有使用applyResourceMapping编译。
         */
        ignoreWarning = false

        /**
         * 运行过程中需要验证基准apk包与补丁包的签名是否一致，是否需要签名。
         */
        useSign = true

        /**
         * optional，default 'true'
         * whether use tinker to build
         */
        tinkerEnable = buildWithTinker()

        /**
         * 编译相关的配置项
         */
        buildConfig {
            /**
             * 可选参数；在编译新的apk时候，我们希望通过保持旧apk的proguard混淆方式，从而减少补丁包的大小。
             * 这个只是推荐设置，不设置applyMapping也不会影响任何的assemble编译。
             */
            applyMapping = getApplyMappingPath()
            /**
             * 可选参数；在编译新的apk时候，我们希望通过旧apk的R.txt文件保持ResId的分配。
             * 这样不仅可以减少补丁包的大小，同时也避免由于ResId改变导致remote view异常。
             */
            applyResourceMapping = getApplyResourceMappingPath()

            /**
             * 在运行过程中，我们需要验证基准apk包的tinkerId是否等于补丁包的tinkerId。
             * 这个是决定补丁包能运行在哪些基准包上面，一般来说我们可以使用git版本号、versionName等等。
             */
            tinkerId = getTinkerIdValue()

            /**
             * 如果我们有多个dex,编译补丁时可能会由于类的移动导致变更增多。
             * 若打开keepDexApply模式，补丁包将根据基准包的类分布来编译。
             */
            keepDexApply = false
        }
        /**
         * dex相关的配置项
         */
        dex {
            /**
             * 只能是'raw'或者'jar'。
             * 对于'raw'模式，我们将会保持输入dex的格式。
             * 对于'jar'模式，我们将会把输入dex重新压缩封装到jar。
             * 如果你的minSdkVersion小于14，你必须选择‘jar’模式，而且它更省存储空间，但是验证md5时比'raw'模式耗时。
             * 默认我们并不会去校验md5,一般情况下选择jar模式即可。
             */
            dexMode = "jar"

            /**
             * 需要处理dex路径，支持*、?通配符，必须使用'/'分割。路径是相对安装包的，例如assets/...
             */
            pattern = ["classes*.dex",
                       "assets/secondary-dex-?.jar"]
            /**
             * 这一项非常重要，它定义了哪些类在加载补丁包的时候会用到。
             * 这些类是通过Tinker无法修改的类，也是一定要放在main dex的类。
             * 这里需要定义的类有：
             * 1. 你自己定义的Application类；
             * 2. Tinker库中用于加载补丁包的部分类，即com.tencent.tinker.loader.*；
             * 3. 如果你自定义了TinkerLoader，需要将它以及它引用的所有类也加入loader中；
             * 4. 其他一些你不希望被更改的类，例如Sample中的BaseBuildInfo类。
             *    这里需要注意的是，这些类的直接引用类也需要加入到loader中。
             *    或者你需要将这个类变成非preverify。
             * 5. 使用1.7.6版本之后版本，参数1、2会自动填写。
             *
             */
            loader = [
                    // Tinker库中用于加载补丁包的部分类
                    "com.tencent.tinker.loader.*",
                    // 自己定义的Application类；
                    "com.tinker.app.AppContext",
                    //use sample, let BaseBuildInfo unchangeable with tinker
                    "tinker.sample.android.app.BaseBuildInfo"
            ]
        }
        /**
         * lib相关的配置项
         */
        lib {
            /**
             * 需要处理lib路径，支持*、?通配符，必须使用'/'分割。
             * 与dex.pattern一致, 路径是相对安装包的，例如assets/...
             */
            pattern = ["lib/*/*.so"]
        }
        /**
         * res相关的配置项
         */
        res {
            /**
             * 需要处理res路径，支持*、?通配符，必须使用'/'分割。
             * 与dex.pattern一致, 路径是相对安装包的，例如assets/...，
             * 务必注意的是，只有满足pattern的资源才会放到合成后的资源包。
             */
            pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]

            /**
             * 支持*、?通配符，必须使用'/'分割。若满足ignoreChange的pattern，在编译时会忽略该文件的新增、删除与修改。
             * 最极端的情况，ignoreChange与上面的pattern一致，即会完全忽略所有资源的修改。
             */
            ignoreChange = ["assets/sample_meta.txt"]

            /**
             * 对于修改的资源，如果大于largeModSize，我们将使用bsdiff算法。
             * 这可以降低补丁包的大小，但是会增加合成时的复杂度。默认大小为100kb
             */
            largeModSize = 100
        }
        /**
         * 用于生成补丁包中的'package_meta.txt'文件
         */
        packageConfig {
            /**
             * configField("key", "value"), 默认我们自动从基准安装包与新安装包的Manifest中读取tinkerId,并自动写入configField。
             * 在这里，你可以定义其他的信息，在运行时可以通过TinkerLoadResult.getPackageConfigByName得到相应的数值。
             * 但是建议直接通过修改代码来实现，例如BuildConfig。
             */
            configField("patchMessage", "tinker is sample to use")
            /**
             * just a sample case, you can use such as sdkVersion, brand, channel...
             * you can parse it in the SamplePatchListener.
             * Then you can use patch conditional!
             */
            configField("platform", "all")
            /**
             * 配置patch补丁版本
             */
            configField("patchVersion", "1.0.0")
        }
        /**
         * 7zip路径配置项，执行前提是useSign为true
         */
        sevenZip {
            /**
             * 例如"com.tencent.mm:SevenZip:1.1.10"，将自动根据机器属性获得对应的7za运行文件，推荐使用。
             */
            zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
            /**
             * 系统中的7za路径，例如"/usr/local/bin/7za"。path设置会覆盖zipArtifact，若都不设置，将直接使用7za去尝试。
             */
            // path = "/usr/local/bin/7za"
        }
    }

```

备份apk到bak文件夹下
```groovy
    /**
    * bak apk and mapping
    */
    android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = variant.name
        def date = new Date().format("MMdd-HH-mm-ss")

        tasks.all {
            if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

                it.doLast {
                    copy {
                        def fileNamePrefix = "${project.name}-${variant.baseName}"
                        def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

                        def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath
                        from variant.outputs.outputFile
                        into destPath
                        rename { String fileName ->
                            fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
                        }

                        from "${buildDir}/outputs/mapping/${variant.dirName}/mapping.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
                        }

                        from "${buildDir}/intermediates/symbols/${variant.dirName}/R.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
                        }
                    }
                }
            }
        }
    }
```


  [1]: http://on8vjlgub.bkt.clouddn.com/simple-project.png "simple-project"
  [2]: http://on8vjlgub.bkt.clouddn.com/assembleRelease.png "assembleRelease"
  [3]: http://on8vjlgub.bkt.clouddn.com/app-release-0314-10-10-21.png "app-release-0314-10-10-21"
  [4]: http://on8vjlgub.bkt.clouddn.com/baseApk_install.png "baseApk_install"
  [5]: http://on8vjlgub.bkt.clouddn.com/device-baseApk.png "device-baseApk"
  [6]: http://on8vjlgub.bkt.clouddn.com/app_build_gradle_ext.png "app_build_gradle_ext"
  [7]: http://on8vjlgub.bkt.clouddn.com/tinkerPatchRelease.png "tinkerPatchRelease"
  [8]: http://on8vjlgub.bkt.clouddn.com/outapk.png "outapk"
  [9]: http://on8vjlgub.bkt.clouddn.com/aftermodifyapk.png "aftermodifyapk"
  [10]: http://on8vjlgub.bkt.clouddn.com/multidex.png "multidex"