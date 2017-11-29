title: 热更新Tinker研究（五）：Application的隔离
date: 2017/04/20 13:44:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

#  热更新Tinker研究（五）：Application的隔离
由于程序默认会加载Application类，所以框架的补丁将不能对它修改了。但是实际过程中却可能需要修改Application中的某些功能。

## 隔离Application
tinker采用的方案是，将原来的Application类隔离起来，即其他任何类都不能再引用我们自己的Application。需要使用到Application类功能的地方采用ApplicationLike来代理。

![enter description here][1]

- 将我们自己Application类以及它的继承类的所有代码拷贝到自己的ApplicationLike继承类中，例如SampleApplicationLike。你也可以直接将自己的Application改为继承ApplicationLike;
- Application的attachBaseContext方法实现要单独移动到onBaseContextAttached中；
- 对ApplicationLike中，引用application的地方改成getApplication();
- 对其他引用Application或者它的静态对象与方法的地方，改成引用ApplicationLike的静态对象与方法；

## 生成目标Application
TinkerApplication会对一些方法作代理，
```java
public abstract class TinkerApplication extends Application {
    ......
    private void onBaseContextAttached(Context base) {
        applicationStartElapsedTime = SystemClock.elapsedRealtime();
        applicationStartMillisTime = System.currentTimeMillis();
        loadTinker();
        ensureDelegate();
        applicationLike.onBaseContextAttached(base);
        //reset save mode
        if (useSafeMode) {
            String processName = ShareTinkerInternals.getProcessName(this);
            String preferName = ShareConstants.TINKER_OWN_PREFERENCE_CONFIG + processName;
            SharedPreferences sp = getSharedPreferences(preferName, Context.MODE_PRIVATE);
            sp.edit().putInt(ShareConstants.TINKER_SAFE_MODE_COUNT, 0).commit();
        }
    }

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        Thread.setDefaultUncaughtExceptionHandler(new TinkerUncaughtHandler(this));
        onBaseContextAttached(base);
    }

    private void loadTinker() {
        //disable tinker, not need to install
        if (tinkerFlags == TINKER_DISABLE) {
            return;
        }
        tinkerResultIntent = new Intent();
        try {
            //reflect tinker loader, because loaderClass may be define by user!
            Class<?> tinkerLoadClass = Class.forName(loaderClassName, false, getClassLoader());

            Method loadMethod = tinkerLoadClass.getMethod(TINKER_LOADER_METHOD, TinkerApplication.class, int.class, boolean.class);
            Constructor<?> constructor = tinkerLoadClass.getConstructor();
            tinkerResultIntent = (Intent) loadMethod.invoke(constructor.newInstance(), this, tinkerFlags, tinkerLoadVerifyFlag);
        } catch (Throwable e) {
            //has exception, put exception error code
            ShareIntentUtil.setIntentReturnCode(tinkerResultIntent, ShareConstants.ERROR_LOAD_PATCH_UNKNOWN_EXCEPTION);
            tinkerResultIntent.putExtra(INTENT_PATCH_EXCEPTION, e);
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        ensureDelegate();
        applicationLike.onCreate();
    }

    @Override
    public void onTerminate() {
        super.onTerminate();
        if (applicationLike != null) {
            applicationLike.onTerminate();
        }
    }

    @Override
    public void onLowMemory() {
        super.onLowMemory();
        if (applicationLike != null) {
            applicationLike.onLowMemory();
        }
    }

    @TargetApi(14)
    @Override
    public void onTrimMemory(int level) {
        super.onTrimMemory(level);
        if (applicationLike != null) {
            applicationLike.onTrimMemory(level);
        }
    }

    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        if (applicationLike != null) {
            applicationLike.onConfigurationChanged(newConfig);
        }
    }

    @Override
    public Resources getResources() {
        Resources resources = super.getResources();
        if (applicationLike != null) {
            return applicationLike.getResources(resources);
        }
        return resources;
    }

    @Override
    public ClassLoader getClassLoader() {
        ClassLoader classLoader = super.getClassLoader();
        if (applicationLike != null) {
            return applicationLike.getClassLoader(classLoader);
        }
        return classLoader;
    }

    @Override
    public AssetManager getAssets() {
        AssetManager assetManager = super.getAssets();
        if (applicationLike != null) {
            return applicationLike.getAssets(assetManager);
        }
        return assetManager;
    }

    @Override
    public Object getSystemService(String name) {
        Object service = super.getSystemService(name);
        if (applicationLike != null) {
            return applicationLike.getSystemService(name, service);
        }
        return service;
    }
    ......
}
```

设置好的application通过AnnotationProcessor生成目标application，生成的类继承TinkerApplication,模板如下
```
package %PACKAGE%;

import com.tencent.tinker.loader.app.TinkerApplication;

/**
 *
 * Generated application for tinker life cycle
 *
 */
public class %APPLICATION% extends TinkerApplication {

    public %APPLICATION%() {
        super(%TINKER_FLAGS%, "%APPLICATION_LIFE_CYCLE%", "%TINKER_LOADER_CLASS%", %TINKER_LOAD_VERIFY_FLAG%);
    }

}
```
然后会对其中的变量进行替换，
```java
            final InputStream is = AnnotationProcessor.class.getResourceAsStream(APPLICATION_TEMPLATE_PATH);
            final Scanner scanner = new Scanner(is);
            final String template = scanner.useDelimiter("\\A").next();
            final String fileContent = template
                .replaceAll("%PACKAGE%", applicationPackageName)
                .replaceAll("%APPLICATION%", applicationClassName)
                .replaceAll("%APPLICATION_LIFE_CYCLE%", lifeCyclePackageName + "." + lifeCycleClassName)
                .replaceAll("%TINKER_FLAGS%", "" + ca.flags())
                .replaceAll("%TINKER_LOADER_CLASS%", "" + loaderClassName)
                .replaceAll("%TINKER_LOAD_VERIFY_FLAG%", "" + ca.loadVerifyFlag());
```
其中会自定义注解，能够让你制定一些属性
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
@Inherited
public @interface DefaultLifeCycle {

    String application();  //指定appliction的className

    String loaderClass() default "com.tencent.tinker.loader.TinkerLoader";   //指定loader class

    int flags();  //指定flag，主要用于控制tinker的开关

    boolean loadVerifyFlag() default false;   //load前是否验证开关
}
```


  [1]: http://on8vjlgub.bkt.clouddn.com/ApplicationLike.png "ApplicationLike"
