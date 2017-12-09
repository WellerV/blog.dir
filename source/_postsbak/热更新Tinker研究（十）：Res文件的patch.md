---

title: 热更新Tinker研究（十）：Res文件的patch
date: 2017/04/20 14:31:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---
 
## 一、解析res_meta.txt
 一个简单的例子如下
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
第一行的条目是title，arscBaseCrc，arscBaseCrc依次按照逗号隔开。pattern是过滤目录，与build.gradle中设置的过滤目录一致。后面再是修改的文件和增加的文件数目和路径。

与res_meta对应的类如下：
```java
public class ShareResPatchInfo {
    public String arscBaseCrc = null;

    public String                         resArscMd5  = null;
    public ArrayList<String>              addRes      = new ArrayList<>();
    public ArrayList<String>              deleteRes   = new ArrayList<>();
    public ArrayList<String>              modRes      = new ArrayList<>();
    //use linkHashMap instead?
    public ArrayList<String>              largeModRes = new ArrayList<>();
    public HashMap<String, LargeModeInfo> largeModMap = new HashMap<>();

    public HashSet<Pattern> patterns = new HashSet<>();
    ...
}
```

变化信息主要有以下几种：
```java
    public static final String RES_ADD_TITLE       = "add:";
    public static final String RES_MOD_TITLE       = "modify:";
    public static final String RES_LARGE_MOD_TITLE = "large modify:";
    public static final String RES_DEL_TITLE       = "delete:";
    public static final String RES_PATTERN_TITLE   = "pattern:";
```

## 二、largeModRes特殊处理
### patch.xml
在对于大的资源文件的patch也不是xml也不是存放所有的xml信息，而是差异化处理的信息。
这里占坑，在生成patch的过程中去研究。

对于大的资源文件的改动，会进行快速复制的方法，
```java
 long largeStart = System.currentTimeMillis();
                ShareResPatchInfo.LargeModeInfo largeModeInfo = resPatchInfo.largeModMap.get(name);

                if (largeModeInfo == null) {
                    TinkerLog.w(TAG, "resource not found largeModeInfo, type:%s, name: %s", ShareTinkerInternals.getTypeString(type), name);
                    manager.getPatchReporter().onPatchPackageCheckFail(patchFile, BasePatchInternal.getMetaCorruptedCode(type));
                    return false;
                }

                largeModeInfo.file = new File(directory, name);
                SharePatchFileUtil.ensureFileDirectory(largeModeInfo.file);

                //we do not check the intermediate files' md5 to save time, use check whether it is 32 length
                if (!SharePatchFileUtil.checkIfMd5Valid(largeModeInfo.md5)) {
                    TinkerLog.w(TAG, "resource meta file md5 mismatch, type:%s, name: %s, md5: %s", ShareTinkerInternals.getTypeString(type), name, largeModeInfo.md5);
                    manager.getPatchReporter().onPatchPackageCheckFail(patchFile, BasePatchInternal.getMetaCorruptedCode(type));
                    return false;
                }
                patchZipFile = new ZipFile(patchFile);
                ZipEntry patchEntry = patchZipFile.getEntry(name);
                if (patchEntry == null) {
                    TinkerLog.w(TAG, "large mod patch entry is null. path:" + name);
                    manager.getPatchReporter().onPatchTypeExtractFail(patchFile, largeModeInfo.file, name, type);
                    return false;
                }

                ZipEntry baseEntry = apkFile.getEntry(name);
                if (baseEntry == null) {
                    TinkerLog.w(TAG, "resources apk entry is null. path:" + name);
                    manager.getPatchReporter().onPatchTypeExtractFail(patchFile, largeModeInfo.file, name, type);
                    return false;
                }
                InputStream oldStream = null;
                InputStream newStream = null;
                try {
                    oldStream = apkFile.getInputStream(baseEntry);
                    newStream = patchZipFile.getInputStream(patchEntry);
                    //采用BSPatch中的patchFast方法  但是会占用更多的内存
                    BSPatch.patchFast(oldStream, newStream, largeModeInfo.file);
                } finally {
                    SharePatchFileUtil.closeQuietly(oldStream);
                    SharePatchFileUtil.closeQuietly(newStream);
                }
                //go go go bsdiff get the
                //md5校验
                if (!SharePatchFileUtil.verifyFileMd5(largeModeInfo.file, largeModeInfo.md5)) {
                    TinkerLog.w(TAG, "Failed to recover large modify file:%s", largeModeInfo.file.getPath());
                    SharePatchFileUtil.safeDeleteFile(largeModeInfo.file);
                    manager.getPatchReporter().onPatchTypeExtractFail(patchFile, largeModeInfo.file, name, type);
                    return false;
                }
                TinkerLog.w(TAG, "success recover large modify file:%s, file size:%d, use time:%d", largeModeInfo.file.getPath(), largeModeInfo.file.length(), (System.currentTimeMillis() - largeStart));
            }
```
关于快速复制



## 三、将所有资源压缩到一个apk中
通过遍历oldApk中文件，只要是符合patterns而且不是被删除，修改以及mainifeset文件，就需要解压出来。然后再依次把mainifeset，修改的大文件，增加的资源文件，添加的资源文件压缩到apk中。

```java
   TinkerZipOutputStream out = null;
            TinkerZipFile oldApk = null;
            TinkerZipFile newApk = null;
            int totalEntryCount = 0;
            try {
                out = new TinkerZipOutputStream(new BufferedOutputStream(new FileOutputStream(resOutput)));
                oldApk = new TinkerZipFile(apkPath);
                newApk = new TinkerZipFile(patchFile);
                final Enumeration<? extends TinkerZipEntry> entries = oldApk.entries();
                while (entries.hasMoreElements()) {
                    TinkerZipEntry zipEntry = entries.nextElement();
                    if (zipEntry == null) {
                        throw new TinkerRuntimeException("zipEntry is null when get from oldApk");
                    }
                    String name = zipEntry.getName();
                    if (name.contains("../")) {
                        continue;
                    }
                    //是否在过滤目录内
                    if (ShareResPatchInfo.checkFileInPattern(resPatchInfo.patterns, name)) {
                        //won't contain in add set.
                        //直接添加 保留的资源
                        if (!resPatchInfo.deleteRes.contains(name)
                            && !resPatchInfo.modRes.contains(name)
                            && !resPatchInfo.largeModRes.contains(name)
                            && !name.equals(ShareConstants.RES_MANIFEST)) {
                            ResUtil.extractTinkerEntry(oldApk, zipEntry, out);
                            totalEntryCount++;
                        }
                    }
                }

                //process manifest
                //提取一个旧的AndroidManifest.xml
                TinkerZipEntry manifestZipEntry = oldApk.getEntry(ShareConstants.RES_MANIFEST);
                if (manifestZipEntry == null) {
                    TinkerLog.w(TAG, "manifest patch entry is null. path:" + ShareConstants.RES_MANIFEST);
                    manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, ShareConstants.RES_MANIFEST, type);
                    return false;
                }
                ResUtil.extractTinkerEntry(oldApk, manifestZipEntry, out);
                totalEntryCount++;

                //大文件作为压缩格式添加
                for (String name : resPatchInfo.largeModRes) {
                    TinkerZipEntry largeZipEntry = oldApk.getEntry(name);
                    if (largeZipEntry == null) {
                        TinkerLog.w(TAG, "large patch entry is null. path:" + name);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type);
                        return false;
                    }
                    ShareResPatchInfo.LargeModeInfo largeModeInfo = resPatchInfo.largeModMap.get(name);
                    ResUtil.extractLargeModifyFile(largeZipEntry, largeModeInfo.file, largeModeInfo.crc, out);
                    totalEntryCount++;
                }

                //增加资源添加
                for (String name : resPatchInfo.addRes) {
                    TinkerZipEntry addZipEntry = newApk.getEntry(name);
                    if (addZipEntry == null) {
                        TinkerLog.w(TAG, "add patch entry is null. path:" + name);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type);
                        return false;
                    }
                    ResUtil.extractTinkerEntry(newApk, addZipEntry, out);
                    totalEntryCount++;
                }

                //修改的资源添加
                for (String name : resPatchInfo.modRes) {
                    TinkerZipEntry modZipEntry = newApk.getEntry(name);
                    if (modZipEntry == null) {
                        TinkerLog.w(TAG, "mod patch entry is null. path:" + name);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type);
                        return false;
                    }
                    ResUtil.extractTinkerEntry(newApk, modZipEntry, out);
                    totalEntryCount++;
                }
            } finally {
                if (out != null) {
                    out.close();
                }
                if (oldApk != null) {
                    oldApk.close();
                }
                if (newApk != null) {
                    newApk.close();
                }
                //delete temp files
                for (ShareResPatchInfo.LargeModeInfo largeModeInfo : resPatchInfo.largeModMap.values()) {
                    SharePatchFileUtil.safeDeleteFile(largeModeInfo.file);
                }
            }
            boolean result = SharePatchFileUtil.checkResourceArscMd5(resOutput, resPatchInfo.resArscMd5);

            if (!result) {
                TinkerLog.i(TAG, "check final new resource file fail path:%s, entry count:%d, size:%d", resOutput.getAbsolutePath(), totalEntryCount, resOutput.length());
                SharePatchFileUtil.safeDeleteFile(resOutput);
                manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, ShareConstants.RES_NAME, type);
                return false;
            }

            TinkerLog.i(TAG, "final new resource file:%s, entry count:%d, size:%d", resOutput.getAbsolutePath(), totalEntryCount, resOutput.length());
        } catch (Throwable e) {
//            e.printStackTrace();
            throw new TinkerRuntimeException("patch " + ShareTinkerInternals.getTypeString(type) +  " extract failed (" + e.getMessage() + ").", e);
        }
```
