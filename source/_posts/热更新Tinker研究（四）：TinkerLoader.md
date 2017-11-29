---

title: 热更新Tinker研究（四）：TinkerLoader
date: 2017/04/20 11:46:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

合成补丁后如何在启动后对应用进行更改呢，处理这个事情的主要类是TinkerLoader,对应dex、res、so文件分别是TinkerDexLoader，TinkerResourceLoader以及TinkerSoLoader。

## 一、data目录下的tinker相关文件
为了更好地控制热更新的过程，以及保存热更新后的结果内容，需要在data目录下保存一些信息和生成patch结果产物。以下是data/data/packageName/下的目录树
```
 .
├── cache
├── code_cache
│   └── com.android.opengl.shaders_cache
├── program_cache
├── shared_prefs
│   └── tinker_own_config_tinker.sample.android.xml
├── tinker
│   ├── info.lock
│   ├── patch-f7435e89
│   │   ├── dex
│   │   │   ├── classes.dex.jar
│   │   │   └── test.dex.jar
│   │   ├── lib
│   │   │   └── lib
│   │   │       ├── arm64-v8a
│   │   │       │   └── libHelloJNI.so
│   │   │       ├── armeabi
│   │   │       │   └── libHelloJNI.so
│   │   │       ├── armeabi-v7a
│   │   │       │   └── libHelloJNI.so
│   │   │       └── x86
│   │   │           └── libHelloJNI.so
│   │   ├── odex
│   │   │   ├── classes.dex.dex
│   │   │   └── test.dex.dex
│   │   ├── patch-f7435e89.apk
│   │   └── res
│   │       └── resources.apk
│   └── patch.info
└── tinker_temp
    └── patch.retry

```
其中shared_prefs目录下文件的作用和sharedpreference的作用类似，用于保存键值对。tinker/patch.info主要保存的printfinger的信息，也就是设备相关信息和一些文件的md5值。/tinker_temp/patch.retry保存的是一些重试信息。tinker/patch-XXX表示某个版本的patch文件，下面有新生成的dex，so,以及resources.apk等。

### 文件示例
#### tinker/patch.info
该文件主要保存升级信息
```txt
#from old version:f7435e89150d55742dfdfc0d3bd52e38 to new version:f7435e89150d55742dfdfc0d3bd52e38
#Thu Apr 06 14:30:35 GMT+08:00 2017
print=GIONEE/GN8002/GIONEE_BBL7516A\:6.0/MRA58K/1466615086\:eng/release-keys
new=f7435e89150d55742dfdfc0d3bd52e38
old=f7435e89150d55742dfdfc0d3bd52e38
```

#### tinker_temp/patch.retry
该文件保存重试信息
```txt
#Thu Apr 06 14:30:22 GMT+08:00 2017
times=2
md5=f7435e89150d55742dfdfc0d3bd52e38
```

#### shared_prefs/tinker_own_config_tinker.sample.android.xml
该文件的作用类似于Android的sharedpreference,用xml保存键值对。
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="safe_mode_count" value="0" />
</map>
```

### 版本控制
在加载补丁目录时，需要根据patch.info,也就是new的md5的前8个字符。
```java
    public static String getPatchVersionDirectory(String version) {
        if (version == null || version.length() != ShareConstants.MD5_LENGTH) {
            return null;
        }

        return ShareConstants.PATCH_BASE_NAME + version.substring(0, 8);
    }
```

## 二、Dex的load
tinker是一种典型的java派别的热修复做法，典型可以参考[
Qzone热更新方案][1]
 ![enter description here][2]
 原理就是去修改dexElements，将new.dex插入到前面，由于在查找类时，会顺序查找，这样就达到了热修复的目的。具体可见[安卓App热补丁动态修复技术介绍][3]。不过Tinker目前的new.dex是一个**全量使用**的情况，也就是直接把所有dex文件都拿过来，所以会有空间占用比较大的情况。不过这种保守的做法可以避免CLASS_ISPREVERIFIED标记校验问题以及插桩引起的内存地址混乱问题。同时在art虚拟机中，dex2oat已经将类的各个地址写死，所以采用插桩方式很可能会导致地址混乱。所以qZone方式既有性能问题，也可能导致错误。
![enter description here][4]
dex的加载主要由TinkerDexLoader负责，除了利用反射加载dex以外，如果系统进行了OTA升级，还会进行dex的优化。

### installDexes
这里动态加载dex的基本原理是利用反射，但是由于不同版本的PathClassLoader机制不一样，这里也需要分版本处理：

#### v23、v19、v14
这三个版本中核心原理都是去修改BaseDexClassLoader中的dexList（DexPathList类型）,PatchClassLoader继承BaseClassLoader，BaseClassLoader继承ClassLoader。
实际上也即是修改DexPathList中dexElements,
```java
/*package*/ final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String zipSeparator = "!/";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private Element[] dexElements;
    ......
}
```
这三个版本不同的地方也只是在不同版本之间，makePathElements方法的查找参数不同。


#### v4 (API 4-13)
v4中的PatchClassLoader
```java
public class PathClassLoader extends ClassLoader {

    private final String path;
    private final String libPath;

    /*
     * Parallel arrays for jar/apk files.
     *
     * (could stuff these into an object and have a single array;       * improves clarity but adds overhead)
     */
    private final String[] mPaths;
    private final File[] mFiles;
    private final ZipFile[] mZips;
    private final DexFile[] mDexs;
    
    ......
}
```
这里主要通过mPaths、mFiles、mZips、mDexs四个数组来控制dex的动态加载，所以只需要利用反射去改变这四个数组的值即可。
V4.install()核心代码
```java
            ShareReflectUtil.expandFieldArray(loader, "mPaths", extraPaths);
            ShareReflectUtil.expandFieldArray(loader, "mFiles", extraFiles);
            ShareReflectUtil.expandFieldArray(loader, "mZips", extraZips);
            try {
                ShareReflectUtil.expandFieldArray(loader, "mDexs", extraDexs);
            } catch (Exception e) {

            }
```

### AndroidNClassLoader
在Android N以上版本，会采用一种parent classLoader的方式，也就是将originClassLoader作为parent,
```java
@TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
class AndroidNClassLoader extends PathClassLoader {
    static ArrayList oldDexFiles = new ArrayList<>();
    PathClassLoader originClassLoader;
   
    private AndroidNClassLoader(String dexPath, PathClassLoader parent) {
        super(dexPath, parent.getParent());
        originClassLoader = parent;
    }
    ......
    //根据不同的情况使用不同的classLoader
    public Class findClass(String name) throws ClassNotFoundException {
        // loader class use default pathClassloader to load
        if (name != null && name.startsWith("com.tencent.tinker.loader.") && !name.equals("com.tencent.tinker.loader.TinkerTestDexLoad")) {
            return originClassLoader.loadClass(name);
        }
        return super.findClass(name);
    }
    ......
}
```
这里不同的类会才用不同的classLoader，如果需要查找Loader相关类，就会从原始classLoader加载，也就是从baseApk中加载，否则就从新生成的AndroidNClassLoader中加载，也就是从new.dex中加载。引入parent classLoader的目的是因为Android N版本会有混合编译，这里可以让缓存失效，避免地址混乱问题，具体可以看下面。

### Android N 混合编译导致补丁机制失效
[Android N混合编译与对热补丁影响解析][5]

### SystemOTA的影响
对于art平台,ota升级后app的boot image已经改变，也就是缓存的热代码，由于厂商只进行了ClassN的优化，所以这里进行一个全量的dex2oat的优化操作。
```java
    if (isSystemOTA) {
            parallelOTAResult = true;
            parallelOTAThrowable = null;
            Log.w(TAG, "systemOTA, try parallel oat dexes!!!!!");

            TinkerParallelDexOptimizer.optimizeAll(
                legalFiles, optimizeDir,
                new TinkerParallelDexOptimizer.ResultCallback() {
                    long start;

                    @Override
                    public void onStart(File dexFile, File optimizedDir) {
                        start = System.currentTimeMillis();
                        Log.i(TAG, "start to optimize dex:" + dexFile.getPath());
                    }

                    @Override
                    public void onSuccess(File dexFile, File optimizedDir, File optimizedFile) {
                        // Do nothing.
                        Log.i(TAG, "success to optimize dex " + dexFile.getPath() + "use time " + (System.currentTimeMillis() - start));
                    }
                    @Override
                    public void onFailed(File dexFile, File optimizedDir, Throwable thr) {
                        parallelOTAResult = false;
                        parallelOTAThrowable = thr;
                        Log.i(TAG, "fail to optimize dex " + dexFile.getPath() + "use time " + (System.currentTimeMillis() - start));
                    }
                }
            );
            if (!parallelOTAResult) {
                Log.e(TAG, "parallel oat dexes failed");
                intentResult.putExtra(ShareIntentUtil.INTENT_PATCH_EXCEPTION, parallelOTAThrowable);
                ShareIntentUtil.setIntentReturnCode(intentResult, ShareConstants.ERROR_LOAD_PATCH_VERSION_PARALLEL_DEX_OPT_EXCEPTION);
                return false;
            }
        }
```

## 三、res的load
加载resource，实际上也是加载外部的apk中的资源。根本原理也是去利用反射更改控制资源文件的类的字段的值。
### res目录的更改
决定从什么目录去加载资源文件，主要由LoaderApk中mRes来控制。
```java

public final class LoadedApk {

    private static final String TAG = "LoadedApk";

    private final ActivityThread mActivityThread;
    private final ApplicationInfo mApplicationInfo;
    final String mPackageName;
    private final String mAppDir;
    private final String mResDir;     //控制去什么目录加载资源文件
    private final String[] mSharedLibraries;
    private final String mDataDir;
    private final String mLibDir;
    private final File mDataDirFile;
    private final ClassLoader mBaseClassLoader;
    private final boolean mSecurityViolation;
    private final boolean mIncludeCode;
    private final DisplayAdjustments mDisplayAdjustments = new DisplayAdjustments();
    Resources mResources;
    private ClassLoader mClassLoader;
    private Application mApplication;

    private final ArrayMap> mReceivers
        = new ArrayMap>();
    private final ArrayMap> mUnregisteredReceivers
        = new ArrayMap>();
    private final ArrayMap> mServices
        = new ArrayMap>();
    private final ArrayMap> mUnboundServices
        = new ArrayMap>();
 
    int mClientCount = 0;
    ......
}
```
而在某些版本中，可能保存在“android.app.ActivityThread$PackageInfo”中，这里不展开讨论。
在AcitivityThread中，下面这些类分别保存着LoadApk的引用，
```java
public final class ActivityThread {
    /** @hide */
    public static final String TAG = "ActivityThread";
    
    ......

    // These can be accessed by multiple threads; mPackages is the lock.
    // XXX For now we keep around information about all packages we have
    // seen, not removing entries from this map.
    // NOTE: The activity and window managers need to call in to
    // ActivityThread to do things like update resource configurations,
    // which means this lock gets held while the activity and window managers
    // holds their own lock.  Thus you MUST NEVER call back into the activity manager
    // or window manager or anything that depends on them while holding this lock.
    final ArrayMap> mPackages
            = new ArrayMap>();
    final ArrayMap> mResourcePackages
            = new ArrayMap>();
    final ArrayList mRelaunchingActivities
            = new ArrayList();
    Configuration mPendingConfiguration = null;

    private final ResourcesManager mResourcesManager;
    ......
}

```
所以我们要更改res文件的加载目录，只需要去遍历mPackages和mResourcePackages中LoadApk,然后修改它们的mRes字段就可以了。
```java
        for (Field field : new Field[]{packagesFiled, resourcePackagesFiled}) {
            Object value = field.get(currentActivityThread);

            for (Map.Entry> entry
                : ((Map>) value).entrySet()) {
                Object loadedApk = entry.getValue().get();
                if (loadedApk == null) {
                    continue;
                }
                if (externalResourceFile != null) {
                    resDir.set(loadedApk, externalResourceFile);
                }
            }
        }
```

### asset目录的修改
所以资源的加载都是由Resouces类来控制，而关于asset目录，由Rescources类里面的mAssets来控制
```java
public class Resources {
    static final String TAG = "Resources";
    ......
    /*package*/ final AssetManager mAssets;
    ......
}
```
AssetManager代码如下，
```java
public final class AssetManager {
    ......
    /**
     * Add an additional set of assets to the asset manager.  This can be
     * either a directory or ZIP file.  Not for use by applications.  Returns
     * the cookie of the added asset, or 0 on failure.
     * {@hide}
     */
    public final int addAssetPath(String path) {
        int res = addAssetPathNative(path);
        return res;
    }
    ......
}    
```
可以看到只需要增加我们指定的path就可以了，但是由于考虑到第三方ROM,
需要将Resources中mAssets引用的指向修改掉。
```java
        // Baidu os
        if (assets.getClass().getName().equals("android.content.res.BaiduAssetManager")) {
            Class baiduAssetManager = Class.forName("android.content.res.BaiduAssetManager");
            newAssetManager = (AssetManager) baiduAssetManager.getConstructor().newInstance();
        } else {
            newAssetManager = AssetManager.class.getConstructor().newInstance();
        }
```
重新指向mAssets
```java
     // Create a new AssetManager instance and point it to the resources installed under
        if (((Integer) addAssetPathMethod.invoke(newAssetManager, externalResourceFile)) == 0) {
            throw new IllegalStateException("Could not create new AssetManager");
        }

        // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
        // in L, so we do it unconditionally.
        ensureStringBlocksMethod.invoke(newAssetManager);

        for (WeakReference wr : references) {
            Resources resources = wr.get();
            //pre-N
            if (resources != null) {
                // Set the AssetManager of the Resources instance to our brand new one
                try {
                    assetsFiled.set(resources, newAssetManager);
                } catch (Throwable ignore) {
                    // N
                    Object resourceImpl = resourcesImplFiled.get(resources);
                    // for Huawei HwResourcesImpl
                    Field implAssets = ShareReflectUtil.findField(resourceImpl, "mAssets");
                    implAssets.setAccessible(true);
                    implAssets.set(resourceImpl, newAssetManager);
                }

                clearPreloadTypedArrayIssue(resources);

                resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
            }
        }
```

## 四、so的load
so的动态加载比较简单，原理也是利用反射，得到“nativeLibraryDirectories”，最后在libDirs里面加入自定义的folder目录。
```java
   Field nativeLibraryDirectories = ShareReflectUtil.findField(dexPathList, "nativeLibraryDirectories");

            List libDirs = (List) nativeLibraryDirectories.get(dexPathList);
            libDirs.add(0, folder);
            Field systemNativeLibraryDirectories =
                ShareReflectUtil.findField(dexPathList, "systemNativeLibraryDirectories");
            List systemLibDirs = (List) systemNativeLibraryDirectories.get(dexPathList);
            Method makePathElements =
                ShareReflectUtil.findMethod(dexPathList, "makePathElements", List.class, File.class, List.class);
            ArrayList suppressedExceptions = new ArrayList<>();
            libDirs.addAll(systemLibDirs);
```
而在API4-13的版本,主要是获取“libraryPathElements”。
```java
    private static final class V4 {
        private static void install(ClassLoader classLoader, File folder)  throws Throwable {
            String addPath = folder.getPath();
            Field pathField = ShareReflectUtil.findField(classLoader, "libPath");
            StringBuilder libPath = new StringBuilder((String) pathField.get(classLoader));
            libPath.append(':').append(addPath);
            pathField.set(classLoader, libPath.toString());

            Field libraryPathElementsFiled = ShareReflectUtil.findField(classLoader, "libraryPathElements");
            List libraryPathElements = (List) libraryPathElementsFiled.get(classLoader);
            libraryPathElements.add(0, addPath);
            libraryPathElementsFiled.set(classLoader, libraryPathElements);
        }
    }
```

## tinker优缺点分析
### 优点
- 开发透明； 开发者无需关心是否在补丁版本，他可以随意修改，不由框架限制；
- 性能影响较小； 对比市面上其他框架，性能影响较小。
- 完整支持； 支持代码，So 库以及资源的修复，可以发布功能。
- 补丁大小较小； 补丁大小较小，提高升级率。
- 稳定，兼容性好； 微信的数亿用户的使用。
- 可配置性高；框架的很多类可以扩展定制。

### 缺点
- Android N的支持不完美：不同的虚拟机都采用全量补丁，会使AndroidN的混合编译退化，使用了动态加载（实际上全量加载），会对性能有较大影响。
- patch后的空间占用大：由于使用全量补丁，合成后新的文件占用空间比较大。

## tinker未来发展趋势
分平台合成
　![enter description here][6]


  [1]: https://zhuanlan.zhihu.com/p/20308548
  [2]: http://on8vjlgub.bkt.clouddn.com/java-patch.png "java-patch"
  [3]: https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a
  [4]: http://on8vjlgub.bkt.clouddn.com/dexLoad.png "dexLoad"
  [5]: http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=1&srcid=0811lK7OQal4gyEfWwUngZ9L#rd
  [6]: http://oa5504rxk.bkt.clouddn.com/dev_Club_08/9.jpg
