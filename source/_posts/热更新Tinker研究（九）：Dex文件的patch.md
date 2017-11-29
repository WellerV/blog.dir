---

title:  热更新Tinker研究（九）：Dex文件的patch
date: 2017/04/20 14:29:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

#  热更新Tinker研究（九）：Dex文件的patch
本文主要讲解dex文件的patch过程，从tinker的DexPatchFile格式分析，对doFullPatch()作为重点讲解。

doFullPatch()的整个过程如图所示：
![enter description here][39]
## 一、patch文件的dex格式
这里有别于标准的dex文件格式，patch文件中dex主要用于保存patch过程中的变换，比如add、delete、replace的相关信息，而这些信息按照区域来记录。比如有stringIds、typeIds等等，这些区域都独立对应add、delete、replace信息。
具体的数据结构如下图所示： 
![enter description here][1]
这里也可以分为两个区域，一个是header,一个是data区域。
其中magic代表此格式的标识，为DXDIFF，二进制表示
```
0x44, 0x58, 0x44, 0x49, 0x46, 0x46
```
version代表DexPatchFile格式的版本，也就是这个数据结构可能会发生变化。
patchedDexSize表示patch后生成的dex文件的大小。firstChunkOffset表示data区域的起始位置。
后面的patchedXXXOffset表示数据区域每个section的偏移量。
oldDexSignature是dex文件的签名。

tinker中的DexPatchFile如下
```java
public final class DexPatchFile {
    public static final byte[] MAGIC = {0x44, 0x58, 0x44, 0x49, 0x46, 0x46}; // DXDIFF
    public static final short CURRENT_VERSION = 0x0002;
    private final DexDataBuffer buffer;
    private short version;
    private int patchedDexSize;
    private int firstChunkOffset;
    private int patchedStringIdSectionOffset;
    private int patchedTypeIdSectionOffset;
    private int patchedProtoIdSectionOffset;
    private int patchedFieldIdSectionOffset;
    private int patchedMethodIdSectionOffset;
    private int patchedClassDefSectionOffset;
    private int patchedMapListSectionOffset;
    private int patchedTypeListSectionOffset;
    private int patchedAnnotationSetRefListSectionOffset;
    private int patchedAnnotationSetSectionOffset;
    private int patchedClassDataSectionOffset;
    private int patchedCodeSectionOffset;
    private int patchedStringDataSectionOffset;
    private int patchedDebugInfoSectionOffset;
    private int patchedAnnotationSectionOffset;
    private int patchedEncodedArraySectionOffset;
    private int patchedAnnotationsDirectorySectionOffset;
    private byte[] oldDexSignature;
	...
}
```

为了更直观的来看请数据结构，特意解析了一个真实环境的patch的dex文件，以json格式描述，data区域省略。
![enter description here][2]

## 二，流程概述
### 1，解析dex_meta信息
dex_meta主要包含name，destMd5InDvm，destMd5InArt，dexDiffMd5，oldDexCrc等。通过文本信息解析，并保存在List里面。
```java
    public static void parseDexDiffPatchInfo(String meta, ArrayList<ShareDexDiffPatchInfo> dexList) {
        if (meta == null || meta.length() == 0) {
            return;
        }
        String[] lines = meta.split("\n");
        for (final String line : lines) {
            if (line == null || line.length() <= 0) {
                continue;
            }
            final String[] kv = line.split(",", 7);
            if (kv == null || kv.length < 7) {
                continue;
            }

            // key
            final String name = kv[0].trim();
            final String path = kv[1].trim();
            final String destMd5InDvm = kv[2].trim();
            final String destMd5InArt = kv[3].trim();
            final String dexDiffMd5 = kv[4].trim();
            final String oldDexCrc = kv[5].trim();
            final String dexMode = kv[6].trim();

            ShareDexDiffPatchInfo dexInfo = new ShareDexDiffPatchInfo(name, path, destMd5InDvm, destMd5InArt, dexDiffMd5, oldDexCrc, dexMode);
            dexList.add(dexInfo);
        }
    }
```
### 2，分情况patch
取出前面meta信息，然后对每个dex文件进行patch。分以下三种情况，
- oldDex不存在
- oldDex存在，patch中的dex为空
- oldDex存在，且patchDex文件不为空

#### 1） oldDex不存在
如果oldDex不存在，直接按照patch信息重新打包dex。
```java
                if (oldDexCrc.equals("0")) {
                    if (patchFileEntry == null) {
                        TinkerLog.w(TAG, "patch entry is null. path:" + patchRealPath);
                        manager.getPatchReporter().onPatchTypecaozExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }

                    //it is a new file, but maybe we need to repack the dex file
                    if (!extractDexFile(patch, patchFileEntry, extractedFile, info)) {
                        TinkerLog.w(TAG, "Failed to extract raw patch file " + extractedFile.getPath());
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }
                }
```
#### 2） oldDex存在，patch中的dex为空
更新dex不存在，在dalvik vm下直接不处理即可。在art vm下，需要拷贝oldDex。
```java
 else if (dexDiffMd5.equals("0")) {
                    // skip process old dex for real dalvik vm
                    if (!ShareTinkerInternals.isVmArt()) {
                        continue;
                    }

                    if (rawApkFileEntry == null) {
                        TinkerLog.w(TAG, "apk entry is null. path:" + patchRealPath);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }

                    //check source crc instead of md5 for faster
                    String rawEntryCrc = String.valueOf(rawApkFileEntry.getCrc());
                    if (!rawEntryCrc.equals(oldDexCrc)) {
                        TinkerLog.e(TAG, "apk entry %s crc is not equal, expect crc: %s, got crc: %s", patchRealPath, oldDexCrc, rawEntryCrc);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }

                    // Small patched dex generating strategy was disabled, we copy full original dex directly now.
                    //patchDexFile(apk, patch, rawApkFileEntry, null, info, smallPatchInfoFile, extractedFile);
                    extractDexFile(apk, rawApkFileEntry, extractedFile, info);

                    if (!SharePatchFileUtil.verifyDexFileMd5(extractedFile, extractedFileMd5)) {
                        TinkerLog.w(TAG, "Failed to recover dex file when verify patched dex: " + extractedFile.getPath());
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        SharePatchFileUtil.safeDeleteFile(extractedFile);
                        return false;
                    }
                } 
```
#### 3）常规情况
需要对oldDex做crc校验，然后进行patchDexFile操作。
```java
else {
                    if (patchFileEntry == null) {
                        TinkerLog.w(TAG, "patch entry is null. path:" + patchRealPath);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }

                    if (!SharePatchFileUtil.checkIfMd5Valid(dexDiffMd5)) {
                        TinkerLog.w(TAG, "meta file md5 invalid, type:%s, name: %s, md5: %s", ShareTinkerInternals.getTypeString(type), info.rawName, dexDiffMd5);
                        manager.getPatchReporter().onPatchPackageCheckFail(patchFile, BasePatchInternal.getMetaCorruptedCode(type));
                        return false;
                    }

                    if (rawApkFileEntry == null) {
                        TinkerLog.w(TAG, "apk entry is null. path:" + patchRealPath);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }
                    //check source crc instead of md5 for faster
                    String rawEntryCrc = String.valueOf(rawApkFileEntry.getCrc());
                    if (!rawEntryCrc.equals(oldDexCrc)) {
                        TinkerLog.e(TAG, "apk entry %s crc is not equal, expect crc: %s, got crc: %s", patchRealPath, oldDexCrc, rawEntryCrc);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        return false;
                    }

                    patchDexFile(apk, patch, rawApkFileEntry, patchFileEntry, info, extractedFile);

                    if (!SharePatchFileUtil.verifyDexFileMd5(extractedFile, extractedFileMd5)) {
                        TinkerLog.w(TAG, "Failed to recover dex file when verify patched dex: " + extractedFile.getPath());
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, extractedFile, info.rawName, type);
                        SharePatchFileUtil.safeDeleteFile(extractedFile);
                        return false;
                    }

                    TinkerLog.w(TAG, "success recover dex file: %s, size: %d, use time: %d",
                            extractedFile.getPath(), extractedFile.length(), (System.currentTimeMillis() - start));
                }
```
### 3,优化dex文件
在不同虚拟机下，分情况进行dex文件优化。
```java
 final String optimizeDexDirectory = patchVersionDirectory + "/" + DEX_OPTIMIZE_PATH + "/";
            File optimizeDexDirectoryFile = new File(optimizeDexDirectory);

            if (!optimizeDexDirectoryFile.exists() && !optimizeDexDirectoryFile.mkdirs()) {
                TinkerLog.w(TAG, "patch recover, make optimizeDexDirectoryFile fail");
                return false;
            }
            // add opt files
            for (File file : files) {
                String outputPathName = SharePatchFileUtil.optimizedPathFor(file, optimizeDexDirectoryFile);
                optFiles.add(new File(outputPathName));
            }

            TinkerLog.w(TAG, "patch recover, try to optimize dex file count:%d", files.length);

            // only use parallel dex optimizer for art
            if (ShareTinkerInternals.isVmArt()) {
                failOptDexFile.clear();
                // try parallel dex optimizer
                TinkerParallelDexOptimizer.optimizeAll(
                    files, optimizeDexDirectoryFile,
                    new TinkerParallelDexOptimizer.ResultCallback() {
                        long startTime;

                        @Override
                        public void onStart(File dexFile, File optimizedDir) {
                            startTime = System.currentTimeMillis();
                            TinkerLog.i(TAG, "start to parallel optimize dex %s, size: %d", dexFile.getPath(), dexFile.length());
                        }

                        @Override
                        public void onSuccess(File dexFile, File optimizedDir, File optimizedFile) {
                            // Do nothing.
                            TinkerLog.i(TAG, "success to parallel optimize dex %s, opt file size: %d, use time %d",
                                dexFile.getPath(), optimizedFile.length(), (System.currentTimeMillis() - startTime));
                        }

                        @Override
                        public void onFailed(File dexFile, File optimizedDir, Throwable thr) {
                            TinkerLog.i(TAG, "fail to parallel optimize dex %s use time %d",
                                dexFile.getPath(), (System.currentTimeMillis() - startTime));
                            failOptDexFile.add(dexFile);
                        }
                    }
                );
                // try again
                for (File retryDexFile : failOptDexFile) {
                    try {
                        String outputPathName = SharePatchFileUtil.optimizedPathFor(retryDexFile, optimizeDexDirectoryFile);

                        if (!SharePatchFileUtil.isLegalFile(retryDexFile)) {
                            manager.getPatchReporter().onPatchDexOptFail(patchFile, retryDexFile,
                                optimizeDexDirectory, retryDexFile.getName(), new TinkerRuntimeException("retry dex optimize file is not exist, name: " + retryDexFile.getName()));
                            return false;
                        }
                        TinkerLog.i(TAG, "try to retry dex optimize file, path: %s, size: %d", retryDexFile.getPath(), retryDexFile.length());
                        long start = System.currentTimeMillis();
                        DexFile.loadDex(retryDexFile.getAbsolutePath(), outputPathName, 0);

                        TinkerLog.i(TAG, "success retry dex optimize file, path: %s, opt file size: %d, use time: %d",
                            retryDexFile.getPath(), new File(outputPathName).length(), (System.currentTimeMillis() - start));
                    } catch (Throwable e) {
                        TinkerLog.e(TAG, "retry dex optimize or load failed, path:" + retryDexFile.getPath());
                        manager.getPatchReporter().onPatchDexOptFail(patchFile, retryDexFile, optimizeDexDirectory, retryDexFile.getName(), e);
                        return false;
                    }
                }
            // for dalvik, machine hardware performance is much worse than art machine
            } else {
                for (File file : files) {
                    try {
                        String outputPathName = SharePatchFileUtil.optimizedPathFor(file, optimizeDexDirectoryFile);
                        long start = System.currentTimeMillis();
                        DexFile.loadDex(file.getAbsolutePath(), outputPathName, 0);
                        TinkerLog.i(TAG, "success single dex optimize file, path: %s, opt file size: %d, use time: %d", file.getPath(), new File(outputPathName).length(),
                            (System.currentTimeMillis() - start));
                    } catch (Throwable e) {
                        TinkerLog.e(TAG, "single dex optimize or load failed, path:" + file.getPath());
                        manager.getPatchReporter().onPatchDexOptFail(patchFile, file, optimizeDexDirectory, file.getName(), e);
                        return false;
                    }
                }
            }
```

## 三、patchDexFileadjustFieldIdIndex
真正负责patch的类是DexPatchApplier，类的结构如下，
```java
public class DexPatchApplier {
    private final Dex oldDex;	//baseApk中的dex
    private final Dex patchedDex;   //目标生成的dex

    private final DexPatchFile patchFile;  //patch文件中dex

    private final SparseIndexMap oldToPatchedIndexMap;  //oldDex到newDex之间的对应关系记录
adjustFieldIdIndex
//下面是不同section对应的patch计算类
    private DexSectionPatchAlgorithm<StringData> stringDataSectionPatchAlg;
    private DexSectionPatchAlgorithm<Integer> typeIdSectionPatchAlg;
    private DexSectionPatchAlgorithm<ProtoId> protoIdSectionPatchAlg;
    private DexSectionPatchAlgorithm<FieldId> fieldIdSectionPatchAlg;
    private DexSectionPatchAlgorithm<MethodId> methodIdSectionPatchAlg;
    private DexSectionPatchAlgorithm<ClassDef> classDefSectionPatchAlg;
    private DexSectionPatchAlgorithm<TypeList> typeListSectionPatchAlg;
    private DexSectionPatchAlgorithm<AnnotationSetRefList> annotationSetRefListSectionPatchAlg;
    private DexSectionPatchAlgorithm<AnnotationSet> annotationSetSectionPatchAlg;
    private DexSectionPatchAlgorithm<ClassData> classDataSectionPatchAlg;
    private DexSectionPatchAlgorithm<Code> codeSectionPatchAlg;
    private DexSectionPatchAlgorithm<DebugInfoItem> debugInfoSectionPatchAlg;
    private DexSectionPatchAlgorithm<Annotation> annotationSectionPatchAlg;
    private DexSectionPatchAlgorithm<EncodedValue> encodedArraySectionPatchAlg;
    private DexSectionPatchAlgorithm<AnnotationsDirectory> annotationsDirectorySectionPatchAlg;
	...
}
```
DexPatchApplier的executeAndSaveTo()首先会进行签名的校验，然后会分为四个步骤
```flow 
st=>start: 开始
e=>end: 结束
op1=>operation: 设置patch后的section属性
op2=>operation: 每个section独立去做patch操作
op3=>operation: 写入header和map
op4=>operation: 写入文件

st->op1->op2->op3->op4->e
```
部分类关系如下
!adjustFieldIdIndex[enter description here][3]
由于DexPatchFile里面有各个section的offset,利用此属性来进行section属性的设置。
每个section的patch步骤都是相同，这里只需要分析DexSectionPatchAlgorithm中的execute()。
这里先读取DexPatchFile中的变化信息，再做一次fullPatch()。
```java
        final int deletedItemCount = patchFile.getBuffer().readUleb128();
        final int[] deletedIndices = readDeltaIndiciesOrOffsets(deletedItemCount);

        final int addedItemCount = patchFile.getBuffer().readUleb128();
        final int[] addedIndices = readDeltaIndiciesOrOffsets(addedItemCount);

        final int replacedItemCount = patchFile.getBuffer().readUleb128();
        final int[] replacedIndices = readDeltaIndiciesOrOffsets(replacedItemCount);

        final TableOfContents.Section tocSec = getTocSection(this.oldDex);
        Dex.Section oldSection = null;

        int oldItemCount = 0;
        if (tocSec.exists()) {
            oldSection = this.oldDex.openSection(tocSec);
            oldItemCount = tocSec.size;
        }

        // Now rest data are added and replaced items arranged in the order of
        // added indices and replaced indices.
        doFullPatch(
                oldSection, oldItemCount, deletedIndices, addedIndices, replacedIndices
        );
```
### doFullPatch()
这里会计算出两个count，oldItemCount和newItemCount，分别代表oldDex和newDex的size。
然后用两个游标oldIndex和patchedIndex来遍历，如果是新游标需要增加或者替换的内容，直接writePatchedItem写入newDex中。如果遍历到oldIndex，需要删除或者被替换的内容需要做标记。
否则去根据oldToPatchedIndexMap去调整生成一个新的item,然后写入，并且记录下对应关系。
最后再做位置的校验。
```java
    private void doFullPatch(
            Dex.Section oldSection,
            int oldItemCount,
            int[] deletedIndices,
            int[] addedIndices,
            int[] replacedIndices
    ) {
        int deletedItemCount = deletedIndices.length;
        int addedItemCount = addedIndices.length;
        int replacedItemCount = replacedIndices.length;
        int newItemCount = oldItemCount + addedItemCount - deletedItemCount;    //变化数目

        int deletedItemCounter = 0; //删除数目
        int addActionCursor = 0;    //增加  游标
        int replaceActionCursor = 0;    //替换游标

        int oldIndex = 0;   //oldDex 游标
        int patchedIndex = 0;   //patch 游标
        //只要有一个游标没有结束，就要继续遍历
        while (oldIndex < oldItemCount || patchedIndex < newItemCount) {
            //此位置需要增加
            if (addActionCursor < addedItemCount && addedIndices[addActionCursor] == patchedIndex) {
                T addedItem = nextItem(patchFile.getBuffer());
                int patchedOffset = writePatchedItem(addedItem);
                ++addActionCursor;
                ++patchedIndex;
            } else  //此位置需要替换
            if (replaceActionCursor < replacedItemCount && replacedIndices[replaceActionCursor] == patchedIndex) {
                T replacedItem = nextItem(patchFile.getBuffer());
                int patchedOffset = writePatchedItem(replacedItem);
                ++replaceActionCursor;
                ++patchedIndex;
            } else  //此位置需要删除标记
            if (Arrays.binarySearch(deletedIndices, oldIndex) >= 0) {
                T skippedOldItem = nextItem(oldSection); // skip old item.
                markDeletedIndexOrOffset(
                        oldToPatchedIndexMap,
                        oldIndex,
                        getItemOffsetOrIndex(oldIndex, skippedOldItem)
                );
                ++oldIndex;
                ++deletedItemCounter;
            } else  //此位置需要替换标记
            if (Arrays.binarySearch(replacedIndices, oldIndex) >= 0) {
                T skippedOldItem = nextItem(oldSection); // skip old item.
                markDeletedIndexOrOffset(
                        oldToPatchedIndexMap,
                        oldIndex,
                        getItemOffsetOrIndex(oldIndex, skippedOldItem)
                );
                ++oldIndex;
            } else  //还没有遍历结束，需要将剩下的调整到新的位置
            if (oldIndex < oldItemCount) {
                //拿到旧的item 并进行调整
                T oldItem = adjustItem(this.oldToPatchedIndexMap, nextItem(oldSection));

                int patchedOffset = writePatchedItem(oldItem);

                //插入新的对应关系到oldToPatchedIndexMap
                updateIndexOrOffset(
                        this.oldToPatchedIndexMap,
                        oldIndex,
                        getItemOffsetOrIndex(oldIndex, oldItem),
                        patchedIndex,
                        patchedOffset
                );

                ++oldIndex;
                ++patchedIndex;
            }
        }

        //做位置校验
        if (addActionCursor != addedItemCount || deletedItemCounter != deletedItemCount
                || replaceActionCursor != replacedItemCount
        ) {
            throw new IllegalStateException(
                    String.format(
                            "bad patch operation sequence. addCounter: %d, addCount: %d, "
                                    + "delCounter: %d, delCount: %d, "
                                    + "replaceCounter: %d, replaceCount:%d",
                            addActionCursor,
                            addedItemCount,
                            deletedItemCounter,
                            deletedItemCount,
                            replaceActionCursor,
                            replacedItemCount
                    )
            );
        }
    }
```

### 关于adjustItem()
以 adjustFields(ClassData.Field[] fields)为例，需要根据oldIndex中的位置去调整为新的位置。
```java
    @Override
    public int adjustFieldIdIndex(int fieldIndex) {
        int index = fieldIdsMap.indexOfKey(fieldIndex);
        //去查找旧的fieldIndex在迁移map中是否存在
        if (index < 0) {
            //不存在  如果是删除内容  就返回-1
            return (fieldIndex >= 0 && deletedFieldIds.containsKey(fieldIndex) ? -1 : fieldIndex);
        } else {
            //否则返回新的位置
            return fieldIdsMap.valueAt(index);
        }
    }
```
fieldIdsMap可以理解为迁移map，key表示在oldDex中的位置index,value表示在newDex中的位置。
deletedFieldIds用来表示oldDex中被删除或者替换的内容。
  


  [1]: http://on8vjlgub.bkt.clouddn.com/patchDexFile.png "patchDexFile"
  [2]: http://on8vjlgub.bkt.clouddn.com/PatchDexFileJson.png "PatchDexFileJson"
  [3]: http://on8vjlgub.bkt.clouddn.com/dexSectionPatch.png "dexSectionPatch"
  [39]: http://on8vjlgub.bkt.clouddn.com/doFullPatch.png "doFullPatch"
