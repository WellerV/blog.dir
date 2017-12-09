---

title:  热更新Tinker研究（六）：TinkerPatchPlugin
date: 2017/04/20 13:58:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

在我们运行tinkerPatchDebug或者tinkerPatchRelease任务的时候，会执行TinkerPatchPlugin的apply()，实际上编写一个gradle的task只需要继承Plugin<T>即可。

整个插件做了以下几件事，
```flow 
st=>start: 开始
e=>end: 结束
op1=>operation: 初始化
op2=>operation: 构建TinkerPatchSchemaTask
op3=>operation: 构建TinkerManifestTask
op4=>operation: 构建TinkerResourceIdTask
op5=>operation: 构建proguardConfigTask
op6=>operation: 构建TinkerMultidexConfigTask
op7=>opetation: 给project注入一些任务

st->op1->op2->op3->op4->op5->op6->e
```

整个插件构造的task情况如下图：

![enter description here][1]

部分过程已经省略，左边蓝色的部分是原来就存在的task，右边绿色的task是插件构建的task。

## 一、初始化
### 1，创建一些扩展属性
#### project的扩展tinkerPatch
```groovy
        project.extensions.create('tinkerPatch', TinkerPatchExtension)
```
TinkerPatchExtension的相关源码如下，它的字段就是支持我们去配置的。
```groovy
public class TinkerPatchExtension {
    /**
     * Specifies the old apk path to diff with the new apk
     */
    String oldApk

    /**
     * If there is loader class changes,
     * or Activity, Service, Receiver, Provider change, it will terminal
     * if ignoreWarning is false
     * default: false
     */
    boolean ignoreWarning

    /**
     * If sign the patch file with the android signConfig
     * default: true
     */
    boolean useSign

    /**
     * whether use tinker
     * default: true
     */
    boolean tinkerEnable
	
    ......
}

```

#### 创建tinkerPatch的扩展
tinkerPatch的扩展属性设置如下，
```groovy
        project.tinkerPatch.extensions.create('buildConfig', TinkerBuildConfigExtension, project)

        project.tinkerPatch.extensions.create('dex', TinkerDexExtension, project)
        project.tinkerPatch.extensions.create('lib', TinkerLibExtension)
        project.tinkerPatch.extensions.create('res', TinkerResourceExtension)
        project.tinkerPatch.extensions.create('packageConfig', TinkerPackageConfigExtension, project)
        project.tinkerPatch.extensions.create('sevenZip', TinkerSevenZipExtension, project)
```
具体属性的每个设置类，这里不进行讨论。 

### 2，检测和设置相关属性
#### 检查是否包含com.android.application插件
```groovy
        //判断需要android插件
        if (!project.plugins.hasPlugin('com.android.application')) {
            throw new GradleException('generateTinkerApk: Android Application plugin required')
        }
```
#### 设置jumboMode和preDexLibaries属性
```groovy
            //open jumboMode  增加字符串和方法的容量
            android.dexOptions.jumboMode = true

            //close preDexLibraries
            try {
                android.dexOptions.preDexLibraries = false
            } catch (Throwable e) {
                //no preDexLibraries field, just continue
            }

```
jumboMode设置为true,会允许更多的字符串个数。preDexLibraries会预编译一些libraries,为了更好的提升增量编译的速度，但是clean之后的编译就会变得比较慢。这里preDexLibraries设置为false。

#### 检查是否开启了instantRun
由于插件不支持instantRun,所以这里会进行检测。
```groovy
                def instantRunTask = getInstantRunTask(project, variantName)
                if (instantRunTask != null) {
                    throw new GradleException(
                            "Tinker does not support instant run mode, please trigger build"
                                    + " by assemble${variantName} or disable instant run"
                                    + " in 'File->Settings...'."
                    )
                }
```

## 二、TinkerPatchSchemaTask
TinkerPatchSchemaTask的作用主要是生成patch，由于依赖于assemble任务，会在assemble任务之后执行。这里 InputParam.Builder 会负责接收一系列参数，包含oldApkPath,newApkPath,loaderPattern等等。
```groovy
        InputParam.Builder builder = new InputParam.Builder()
        if (configuration.useSign) {
            if (signConfig == null) {
                throw new GradleException("can't the get signConfig for this build")
            }
            builder.setSignFile(signConfig.storeFile)
                    .setKeypass(signConfig.keyPassword)
                    .setStorealias(signConfig.keyAlias)
                    .setStorepass(signConfig.storePassword)

        }

        builder.setOldApk(configuration.oldApk)
               .setNewApk(buildApkPath)
               .setOutBuilder(outputFolder)
               .setIgnoreWarning(configuration.ignoreWarning)
               .setDexFilePattern(new ArrayList(configuration.dex.pattern))
               .setDexLoaderPattern(new ArrayList(configuration.dex.loader))
               .setDexMode(configuration.dex.dexMode)
               .setSoFilePattern(new ArrayList(configuration.lib.pattern))
               .setResourceFilePattern(new ArrayList(configuration.res.pattern))
               .setResourceIgnoreChangePattern(new ArrayList(configuration.res.ignoreChange))
               .setResourceLargeModSize(configuration.res.largeModSize)
               .setUseApplyResource(configuration.buildConfig.usingResourceMapping)
               .setConfigFields(new HashMap(configuration.packageConfig.getFields()))
               .setSevenZipPath(configuration.sevenZip.path)
               .setUseSign(configuration.useSign)
```
然后由Runner负责具体的生成patch工作，与之相关的类有ApkDecoder、PatchInfo和PatchBuilder。
```java
            //gen patch
            ApkDecoder decoder = new ApkDecoder(config);
            decoder.onAllPatchesStart();
            decoder.patch(config.mOldApkFile, config.mNewApkFile);
            decoder.onAllPatchesEnd();

            //gen meta file and version file
            PatchInfo info = new PatchInfo(config);
            info.gen();

            //build patch
            PatchBuilder builder = new PatchBuilder(config);
            builder.buildPatch();
```
ApkDecoder主要负责对比oldApk和newApk,并生成对应的文件，这里是进行dex,so,res对比，生成差分包的关键步骤buzhou。PatchInfo主要负责由前面得到的信息，来生成package_meta.txt。PatchBuilder负责将patch.apk从tempPath拷贝到最终的路径，并且进行签名，以及利用7-zip去压缩文件。

## 三、TinkerManifestTask
TinkerManifestTask主要是负责处理manifest相关的工作，被processResources所依赖，所以在执行processResources之前会执行TinkerManifestTask。下面是TinkerManifestTask的taskAction的部分源码。
```groovy
    @TaskAction
    def updateManifest() {
        // Parse the AndroidManifest.xml
        String tinkerValue = project.extensions.tinkerPatch.buildConfig.tinkerId
        if (tinkerValue == null || tinkerValue.isEmpty()) {
            throw new GradleException('tinkerId is not set!!!')
        }

        tinkerValue = TINKER_ID_PREFIX + tinkerValue

        project.logger.error("tinker add ${tinkerValue} to your AndroidManifest.xml ${manifestPath}")

        writeManifestMeta(manifestPath, TINKER_ID, tinkerValue)
        addApplicationToLoaderPattern()
        File manifestFile = new File(manifestPath)
        if (manifestFile.exists()) {
            FileOperation.copyFileUsingStream(manifestFile, project.file(MANIFEST_XML))
            project.logger.error("tinker gen AndroidManifest.xml in ${MANIFEST_XML}")
        }

    }
```
这里主要也是做了两件事情，一个是将TINKER_ID写入到manifest,二是将当前的Application类,添加到LoaderPattern。LoaderPattern的作用主要是负责标记哪些类属于Loader主体类，不能用于热更新。

## 四、TinkerResourceIdTask

TinkerResourceIdTask也是在processResources之前执行，主要是负责处理资源文件的id,以保证和baseApk中资源id一致，这样的话在java层的代码dex文件中会做较小的改动。其中控制资源id的两个最重要的文件是ids.xml和public.xml。aapt会根据这两个文件去分配id。
```groovy
  @TaskAction
    def applyResourceId() {
        String resourceMappingFile = project.extensions.tinkerPatch.buildConfig.applyResourceMapping

        // Parse the public.xml and ids.xml
        if (!FileOperation.isLegalFile(resourceMappingFile)) {
            project.logger.error("apply resource mapping file ${resourceMappingFile} is illegal, just ignore")
            return
        }
        String idsXml = resDir + "/values/ids.xml";
        String publicXml = resDir + "/values/public.xml";
        FileOperation.deleteFile(idsXml);
        FileOperation.deleteFile(publicXml);
        List resourceDirectoryList = new ArrayList()
        resourceDirectoryList.add(resDir)

        project.logger.error("we build ${project.getName()} apk with apply resource mapping file ${resourceMappingFile}")
        project.extensions.tinkerPatch.buildConfig.usingResourceMapping = true
        Map> rTypeResourceMap = PatchUtil.readRTxt(resourceMappingFile)

        AaptResourceCollector aaptResourceCollector = AaptUtil.collectResource(resourceDirectoryList, rTypeResourceMap)
        PatchUtil.generatePublicResourceXml(aaptResourceCollector, idsXml, publicXml)
        File publicFile = new File(publicXml)
        if (publicFile.exists()) {
            FileOperation.copyFileUsingStream(publicFile, project.file(RESOURCE_PUBLIC_XML))
            project.logger.error("tinker gen resource public.xml in ${RESOURCE_PUBLIC_XML}")
        }
        File idxFile = new File(idsXml)
        if (idxFile.exists()) {
            FileOperation.copyFileUsingStream(idxFile, project.file(RESOURCE_IDX_XML))
            project.logger.error("tinker gen resource idx.xml in ${RESOURCE_IDX_XML}")
        }
    }

```
首先是根据tinkerPatch中buildConfig中指定的applyResourceMapping文件，applyResourceMapping是baseApk的R.txt的记录。去解析这个文件，然后根据这个规则生成新的ids.xml和public.xml,从而保证资源文件的id变化不大。

## 五、proguardConfigTask
proguardConfigTask在transformClassesAndResourcesWithProguardFor之前执行，它的作用是对于一些特殊混淆规则做处理。
对loader的关键类不进行混淆，
```groovy
    static final String PROGUARD_CONFIG_SETTINGS =
            "-keepattributes *Annotation* \n" +
                    "-dontwarn com.tencent.tinker.anno.AnnotationProcessor \n" +
                    "-keep @com.tencent.tinker.anno.DefaultLifeCycle public class *\n" +
                    "-keep public class * extends android.app.Application {\n" +
                    "    *;\n" +
                    "}\n" +
                    "\n" +
                    "-keep public class com.tencent.tinker.loader.app.ApplicationLifeCycle {\n" +
                    "    *;\n" +
                    "}\n" +
                    "-keep public class * implements com.tencent.tinker.loader.app.ApplicationLifeCycle {\n" +
                    "    *;\n" +
                    "}\n" +
                    "\n" +
                    "-keep public class com.tencent.tinker.loader.TinkerLoader {\n" +
                    "    *;\n" +
                    "}\n" +
                    "-keep public class * extends com.tencent.tinker.loader.TinkerLoader {\n" +
                    "    *;\n" +
                    "}\n" +
                    "-keep public class com.tencent.tinker.loader.TinkerTestDexLoad {\n" +
                    "    *;\n" +
                    "}\n" +
                    "\n"
```
也会对tinkerPatch中dex的loader属性中设置的类进行保留，具体代码如下：
```groovy
  @TaskAction
    def updateTinkerProguardConfig() {
        def file = project.file(PROGUARD_CONFIG_PATH)
        project.logger.error("try update tinker proguard file with ${file}")

        // Create the directory if it doesnt exist already
        file.getParentFile().mkdirs()

        // Write our recommended proguard settings to this file
        FileWriter fr = new FileWriter(file.path)

        String applyMappingFile = project.extensions.tinkerPatch.buildConfig.applyMapping

        //write applymapping
        if (shouldApplyMapping && FileOperation.isLegalFile(applyMappingFile)) {
            project.logger.error("try add applymapping ${applyMappingFile} to build the package")
            fr.write("-applymapping " + applyMappingFile)
            fr.write("\n")
        } else {
            project.logger.error("applymapping file ${applyMappingFile} is illegal, just ignore")
        }

        fr.write(PROGUARD_CONFIG_SETTINGS)

        fr.write("#your dex.loader patterns here\n")
        //they will removed when apply
        Iterable loader = project.extensions.tinkerPatch.dex.loader
        for (String pattern : loader) {
            if (pattern.endsWith("*") && !pattern.endsWith("**")) {
                pattern += "*"
            }
            fr.write("-keep class " + pattern)
            fr.write("\n")
        }
        fr.close()
        // Add this proguard settings file to the list
        applicationVariant.getBuildType().buildType.proguardFiles(file)
        def files = applicationVariant.getBuildType().buildType.getProguardFiles()

        project.logger.error("now proguard files is ${files}")
    }
```

## 六、TinkerMultidexConfigTask 
TinkerMultidexConfigTask在multidexTask之前执行，换句话说，multidexTask是依赖TinkerMultidexConfigTask的。它的主要职责是保证和loader相关的类必须要在mainDex文件中。其原理就是去在maindexlist中添加一些规则，和混淆相关的内容类似，这里也有一个固定的需要keep的类的设置。
```groovy
    static final String MULTIDEX_CONFIG_SETTINGS =
            "-keep public class * implements com.tencent.tinker.loader.app.ApplicationLifeCycle {\n" +
                    "    *;\n" +
                    "}\n" +
                    "\n" +
                    "-keep public class * extends com.tencent.tinker.loader.TinkerLoader {\n" +
                    "    *;\n" +
                    "}\n" +
                    "\n" +
                    "-keep public class * extends android.app.Application {\n" +
                    "    *;\n" +
                    "}\n"

```
然后后面也会写入的配置的loader的相关类，把他们写入到maindexlist中去。
```groovy
 @TaskAction
    def updateTinkerProguardConfig() {
        File file = project.file(MULTIDEX_CONFIG_PATH)
        project.logger.error("try update tinker multidex keep proguard file with ${file}")

        // Create the directory if it doesn't exist already
        file.getParentFile().mkdirs()

        StringBuffer lines = new StringBuffer()
        lines.append("\n")
             .append("#tinker multidex keep patterns:\n")
             .append(MULTIDEX_CONFIG_SETTINGS)
             .append("\n")
             .append("#your dex.loader patterns here\n")

        Iterable loader = project.extensions.tinkerPatch.dex.loader
        for (String pattern : loader) {
            if (pattern.endsWith("*")) {
                if (!pattern.endsWith("**")) {
                    pattern += "*"
                }
            }
            lines.append("-keep class " + pattern + " {\n" +
                    "    *;\n" +
                    "}\n")
                    .append("\n")
        }

        // Write our recommended proguard settings to this file
        FileWriter fr = new FileWriter(file.path)
        try {
            for (String line : lines) {
                fr.write(line)
            }
        } finally {
            fr.close()
        }

        File multiDexKeepProguard = null
        try {
            multiDexKeepProguard = applicationVariant.getVariantData().getScope().getManifestKeepListProguardFile()
        } catch (Throwable ignore) {
            try {
                multiDexKeepProguard = applicationVariant.getVariantData().getScope().getManifestKeepListFile()
            } catch (Throwable e) {
                project.logger.error("can't find getManifestKeepListFile method, exception:${e}")
            }
        }
        if (multiDexKeepProguard == null) {
            project.logger.error("auto add multidex keep pattern fail, you can only copy ${file} to your own multiDex keep proguard file yourself.")
            return
        }
        FileWriter manifestWriter = new FileWriter(multiDexKeepProguard, true)
        try {
            for (String line : lines) {
                manifestWriter.write(line)
            }
        } finally {
            manifestWriter.close()
        }
    }

```

## 七、给project注入一些任务
主要是给transform为DexTransform的TransformTask注入ImmutableDexTransform。ImmutableDexTransform是DexTransform的装饰类，相当于对DexTransform的功能进行补充。
```groovy
      DexTransform dexTransform = task.transform
                            ImmutableDexTransform hookDexTransform = new ImmutableDexTransform(project,
                                    variant, dexTransform)
                            project.logger.info("variant name: " + variant.name)

                            Field field = TransformTask.class.getDeclaredField("transform")
                            field.setAccessible(true)
                            field.set(task, hookDexTransform)
```
这里面主要做的事，包含得到oldDexList,处理mainDexListFile,得到一些映射关系，生成jar，dex文件等等，这里不展开讨论。


  [1]: http://on8vjlgub.bkt.clouddn.com/TinkerPatchPlugin.png "TinkerPatchPlugin"
