---

title:  热更新Tinker研究（七）：Dex的patch文件生成
date: 2017/04/20 14:05:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

 
ApkDecoder中的dexPatchDecoder负责dex的patch生成工作，dexPatchDecoder实际上是UniqueDexDiffDecoder类型。这一系列相关的类的关系如下图所示。
![enter description here][1]
BaseDecoder中有三个抽象方法，onAllPatchesStart（）， patch(File oldFile, File newFile)，onAllPatchesEnd()。其中onAllPatchesStart（）表示patch()进行前，patch()是正式的patch方法，onAllPatchesEnd()用于处理patch完成后的后续工作。
UniqueDexDiffDecoder只是在DexDiffDecoder的基础上，用一个list保存处理过的dex，来保证dex文件的唯一性。

整个过程可以分为以下步骤来讲解
```flow 
st=>start: 开始
e=>end: 结束
op1=>operation: checkIfExcludedClassWasModifiedInNewDex
op2=>operation: collectAddedOrDeletedClasses
op3=>operation: generatePatchInfoFile
op4=>operation: addTestDex


st->op1->op2->op3->op4->e
```
## 一、DexClassesComparator
DexClassesComparator是用于比较dex差异的主要类，其中也几个比较重要的成员变量，
```java
    private final Set<Pattern> patternsOfClassDescToCheck = new HashSet<>();    //检测的集合
    private final Set<Pattern> patternsOfIgnoredRemovedClassDesc = new HashSet<>();     //忽略的集合
```
检测的集合和忽略的集合可以让我们灵活使用，不同的场景下设置不同的参数。
该类比较重要的比较方法具体原理如下，
首先会将类描述（带包名的类名）按照一定的前面的过滤条件放到对应的集合里面，old和new Dex都会处理，
```java
  // Map classDesc and typeIndex to classInfo
        // and collect typeIndex of classes to check in oldDexes.
        for (Dex oldDex : oldDexGroup.dexes) {
            int classDefIndex = 0;
            for (ClassDef oldClassDef : oldDex.classDefs()) {
                String desc = oldDex.typeNames().get(oldClassDef.typeIndex);
                if (Utils.isStringMatchesPatterns(desc, patternsOfClassDescToCheck)) {
                    if (!oldDescriptorOfClassesToCheck.add(desc)) {
                        throw new IllegalStateException(
                                String.format(
                                        "duplicate class descriptor [%s] in different old dexes.",
                                        desc
                                )
                        );
                    }
                }
                DexClassInfo classInfo = new DexClassInfo(desc, classDefIndex, oldClassDef, oldDex);
                ++classDefIndex;
                oldClassDescriptorToClassInfoMap.put(desc, classInfo);
            }
        }

        // Map classDesc and typeIndex to classInfo
        // and collect typeIndex of classes to check in newDexes.
        for (Dex newDex : newDexGroup.dexes) {
            int classDefIndex = 0;
            for (ClassDef newClassDef : newDex.classDefs()) {
                String desc = newDex.typeNames().get(newClassDef.typeIndex);
                if (Utils.isStringMatchesPatterns(desc, patternsOfClassDescToCheck)) {
                    if (!newDescriptorOfClassesToCheck.add(desc)) {
                        throw new IllegalStateException(
                                String.format(
                                        "duplicate class descriptor [%s] in different new dexes.",
                                        desc
                                )
                        );
                    }
                }
                DexClassInfo classInfo = new DexClassInfo(desc, classDefIndex, newClassDef, newDex);
                ++classDefIndex;
                newClassDescriptorToClassInfoMap.put(desc, classInfo);
            }
        }
```
然后再是得到被删除的类，其实就是oldDescriptorOfClassesToCheck -  newDescriptorOfClassesToCheck，差集部分就是我们需要的。
```
        //oldDescriptorOfClassesToCheck -  newDescriptorOfClassesToCheck差集
        Set<String> deletedClassDescs = new HashSet<>(oldDescriptorOfClassesToCheck);
        deletedClassDescs.removeAll(newDescriptorOfClassesToCheck);

        for (String desc : deletedClassDescs) {
            // These classes are deleted as we expect to, so we remove them
            // from result.
            if (Utils.isStringMatchesPatterns(desc, patternsOfIgnoredRemovedClassDesc)) {
                logger.i(TAG, "Ignored deleted class: %s", desc);
                continue;
            } else {
                logger.i(TAG, "Deleted class: %s", desc);
            }
            deletedClassInfoList.add(oldClassDescriptorToClassInfoMap.get(desc));
        }
```
再是找到被增加的类，newDescriptorOfClassesToCheck - oldDescriptorOfClassesToCheck,求差集，
```java
        //被增加的类：newDescriptorOfClassesToCheck - oldDescriptorOfClassesToCheck
        Set<String> addedClassDescs = new HashSet<>(newDescriptorOfClassesToCheck);
        addedClassDescs.removeAll(oldDescriptorOfClassesToCheck);

        for (String desc : addedClassDescs) {
            logger.i(TAG, "Added class: %s", desc);
            addedClassInfoList.add(newClassDescriptorToClassInfoMap.get(desc));
        }
```
要找到被改变的类，首先要找到两个集合的交集，再进行比对，如果发生了改变，就符合被改变的类的条件，加入集合。
```java
  //被改变的类   求交集
        Set<String> mayBeChangedClassDescs = new HashSet<>(oldDescriptorOfClassesToCheck);
        mayBeChangedClassDescs.retainAll(newDescriptorOfClassesToCheck);

        for (String desc : mayBeChangedClassDescs) {
            DexClassInfo oldClassInfo = oldClassDescriptorToClassInfoMap.get(desc);
            DexClassInfo newClassInfo = newClassDescriptorToClassInfoMap.get(desc);
            switch (compareMode) { //？什么区别？
                case COMPARE_MODE_NORMAL: {
                    if (!isSameClass(
                            oldClassInfo.owner,
                            newClassInfo.owner,
                            oldClassInfo.classDef,
                            newClassInfo.classDef
                    )) {
                        logger.i(TAG, "Changed class: %s", desc);
                        changedClassDescToClassInfosMap.put(
                                desc, new DexClassInfo[]{oldClassInfo, newClassInfo}
                        );
                    }
                    break;
                }
                case COMPARE_MODE_CAUSE_REF_CHANGE_ONLY: {
                    if (isClassChangeAffectedToRef(
                            oldClassInfo.owner,
                            newClassInfo.owner,
                            oldClassInfo.classDef,
                            newClassInfo.classDef
                    )) {
                        logger.i(TAG, "Ref-changed class: %s", desc);
                        changedClassDescToClassInfosMap.put(
                                desc, new DexClassInfo[]{oldClassInfo, newClassInfo}
                        );
                    }
                    break;
                }
            }
        }
```

## 二、checkIfExcludedClassWasModifiedInNewDex
主要检测没有被包含差分范围内的类，这里一般就是指设定的loader。这里的检测规则如下，
loader相关类必须存在在oldDex文件中的primary dex,也必须要让它们在新的primary dex中保持一致，
有下面一些情况就会被判断为错误发生：
 - primary old dex找不到
 - primary new dex找不到
 - primary old dex 中不存在loader classes           
 - primary new dex 中存在loader classes
 - loader classes 在primary new dex中不删改
 - loader classes 在secondary old dexes中被发现
 - loader calsses 在secondary new dexes中被发现
 
 相关检测代码如下，主要是利用DexClassesComparator,这里会把loader classes设置为patternsOfClassDescToCheck。
 ```java
    boolean isPrimaryDex = isPrimaryDex((oldFile == null ? newFile : oldFile));

                    if (isPrimaryDex) {
                        if (oldFile == null) {
                            stmCode = STMCODE_ERROR_PRIMARY_OLD_DEX_IS_MISSING;
                        } else if (newFile == null) {
                            stmCode = STMCODE_ERROR_PRIMARY_NEW_DEX_IS_MISSING;
                        } else {
                            dexCmptor.startCheck(oldDex, newDex);
                            deletedClassInfos = dexCmptor.getDeletedClassInfos();
                            addedClassInfos = dexCmptor.getAddedClassInfos();
                            changedClassInfosMap = dexCmptor.getChangedClassDescToInfosMap();

                            // All loader classes are in new dex, while none of them in old one.
                            //这里正确的情况应该是三个infos都为空
                            //如果deletedClassInfos和changedClassInfosMap为空而addedClassInfos不为空，表示loader classes不全在primary old dex中
                            if (deletedClassInfos.isEmpty() && changedClassInfosMap.isEmpty() && !addedClassInfos.isEmpty()) {
                                stmCode = STMCODE_ERROR_LOADER_CLASS_NOT_IN_PRIMARY_OLD_DEX;
                            } else {
                                if (deletedClassInfos.isEmpty() && addedClassInfos.isEmpty()) {
                                    // class descriptor is completely matches, see if any contents changes.
                                    if (changedClassInfosMap.isEmpty()) {
                                        stmCode = STMCODE_END;
                                    } else {
                                        //如果有改变
                                        stmCode = STMCODE_ERROR_LOADER_CLASS_CHANGED;
                                    }
                                } else {
                                    //有删除  或者 增加
                                    stmCode = STMCODE_ERROR_LOADER_CLASS_IN_PRIMARY_DEX_MISMATCH;
                                }
                            }
                        }
                    } else {
                        //后面主要检测loader classes是否存在于secondary dexes
                        Set<Pattern> patternsOfClassDescToCheck = new HashSet<>();
                        for (String patternStr : config.mDexLoaderPattern) {
                            patternsOfClassDescToCheck.add(
                                Pattern.compile(
                                    PatternUtils.dotClassNamePatternToDescriptorRegEx(patternStr)
                                )
                            );
                        }

                        if (oldDex != null) {
                            oldClassesDescToCheck.clear();
                            for (ClassDef classDef : oldDex.classDefs()) {
                                String desc = oldDex.typeNames().get(classDef.typeIndex);
                                if (Utils.isStringMatchesPatterns(desc, patternsOfClassDescToCheck)) {
                                    oldClassesDescToCheck.add(desc);
                                }
                            }
                            if (!oldClassesDescToCheck.isEmpty()) {
                                stmCode = STMCODE_ERROR_LOADER_CLASS_FOUND_IN_SECONDARY_OLD_DEX;
                                break;
                            }
                        }

                        if (newDex != null) {
                            newClassesDescToCheck.clear();
                            for (ClassDef classDef : newDex.classDefs()) {
                                String desc = newDex.typeNames().get(classDef.typeIndex);
                                if (Utils.isStringMatchesPatterns(desc, patternsOfClassDescToCheck)) {
                                    newClassesDescToCheck.add(desc);
                                }
                            }
                            if (!newClassesDescToCheck.isEmpty()) {
                                stmCode = STMCODE_ERROR_LOADER_CLASS_FOUND_IN_SECONDARY_NEW_DEX;
                                break;
                            }
                        }

                        stmCode = STMCODE_END;
                    }
 ```
## 三、collectAddedOrDeletedClasses
![enter description here][2]
这里主要是为了得到增加和删除的classes,和DexClassesComparator中原理类似，也是利用两个集合求差集。
```java
  Dex oldDex = new Dex(oldFile);
        Dex newDex = new Dex(newFile);

        Set<String> oldClassDescs = new HashSet<>();
        for (ClassDef oldClassDef : oldDex.classDefs()) {
            oldClassDescs.add(oldDex.typeNames().get(oldClassDef.typeIndex));
        }

        Set<String> newClassDescs = new HashSet<>();
        for (ClassDef newClassDef : newDex.classDefs()) {
            newClassDescs.add(newDex.typeNames().get(newClassDef.typeIndex));
        }

        // newClassDescs - oldClassDescs得到 增加的类
        Set<String> addedClassDescs = new HashSet<>(newClassDescs);
        addedClassDescs.removeAll(oldClassDescs);

        //oldClassDescs - newClassDescs得到 删除的类
        Set<String> deletedClassDescs = new HashSet<>(oldClassDescs);
        deletedClassDescs.removeAll(newClassDescs);

        for (String addedClassDesc : addedClassDescs) {
            if (addedClassDescToDexNameMap.containsKey(addedClassDesc)) {
                throw new TinkerPatchException(
                        String.format(
                                "Class Duplicate. Class [%s] is added in both new dex: [%s] and [%s]. Please check your newly apk.",
                                addedClassDesc,
                                addedClassDescToDexNameMap.get(addedClassDesc),
                                newFile.toString()
                        )
                );
            } else {
                addedClassDescToDexNameMap.put(addedClassDesc, newFile.toString());
            }
        }

        for (String deletedClassDesc : deletedClassDescs) {
            if (deletedClassDescToDexNameMap.containsKey(deletedClassDesc)) {
                throw new TinkerPatchException(
                        String.format(
                                "Class Duplicate. Class [%s] is deleted in both old dex: [%s] and [%s]. Please check your base apk.",
                                deletedClassDesc,
                                addedClassDescToDexNameMap.get(deletedClassDesc),
                                oldFile.toString()
                        )
                );
            } else {
                deletedClassDescToDexNameMap.put(deletedClassDesc, newFile.toString());
            }
        }

```

## 四、generatePatchInfoFile
### generatePatchedDexInfoFile
根据oldDex和newDex的对应关系，来生成dex_meta.txt中的列表，其中
oldAndNewDexFilePairList保存这oldDex和newDex中的对应关系，具体代码
```java
    private void generatePatchedDexInfoFile() {
        // Generate dex diff out and full patched dex if a pair of dex is different.
        for (AbstractMap.SimpleEntry<File, File> oldAndNewDexFilePair : oldAndNewDexFilePairList) {
            File oldFile = oldAndNewDexFilePair.getKey();
            File newFile = oldAndNewDexFilePair.getValue();
            final String dexName = getRelativeDexName(oldFile, newFile);
            RelatedInfo relatedInfo = dexNameToRelatedInfoMap.get(dexName);
            if (!relatedInfo.oldMd5.equals(relatedInfo.newMd5)) {
                diffDexPairAndFillRelatedInfo(oldFile, newFile, relatedInfo);
            } else {
                // In this case newDexFile is the same as oldDexFile, but we still
                // need to treat it as patched dex file so that the SmallPatchGenerator
                // can analyze which class of this dex should be kept in small patch.
                relatedInfo.newOrFullPatchedFile = newFile;
                relatedInfo.newOrFullPatchedMd5 = relatedInfo.newMd5;
            }
        }
    }
```

### logDexesToDexMeta
这个方法主要是根据前面记录的RelatedInfo，来把dex的patch信息写入dex_meta.txt中，这里会针对不同的情况进行不同的操作，部分代码如下
```java
 for (AbstractMap.SimpleEntry<File, File> oldAndNewDexFilePair : oldAndNewDexFilePairList) {
            final File oldDexFile = oldAndNewDexFilePair.getKey();
            final File newDexFile = oldAndNewDexFilePair.getValue();
            final String dexName = getRelativeDexName(oldDexFile, newDexFile);
            final RelatedInfo relatedInfo = dexNameToRelatedInfoMap.get(dexName);
            if (!relatedInfo.oldMd5.equals(relatedInfo.newMd5)) {
                //logToDexMeta(newDexFile, oldDexFile, relatedInfo.dexDiffFile, relatedInfo.newOrFullPatchedMd5, relatedInfo.smallPatchedMd5, relatedInfo.dexDiffMd5);
                logToDexMeta(newDexFile, oldDexFile, relatedInfo.dexDiffFile, relatedInfo.newOrFullPatchedMd5, relatedInfo.newOrFullPatchedMd5, relatedInfo.dexDiffMd5);
            } else {
                // For class N dexes, if new dex is the same as old dex, we should log it as 'copy directly'
                // in dex meta to fix problems in Art environment.
                if (realClassNDexFiles.contains(oldDexFile)) {
                    //if (!"0".equals(relatedInfo.smallPatchedMd5)) {
                    //    logToDexMeta(newDexFile, oldDexFile, null, "0", relatedInfo.smallPatchedMd5, "0");
                    //}

                    // Bugfix: However, if what we would copy directly is main dex, we should do an additional diff operation
                    // so that patch applier would help us remove all loader classes of it in runtime.
                    if (dexName.equals(DexFormat.DEX_IN_JAR_NAME)) {
                        Logger.d("\nDo additional diff on main dex to remove loader classes in it.");
                        diffDexPairAndFillRelatedInfo(oldDexFile, newDexFile, relatedInfo);
                        logToDexMeta(newDexFile, oldDexFile, relatedInfo.dexDiffFile, relatedInfo.newOrFullPatchedMd5, relatedInfo.newOrFullPatchedMd5, relatedInfo.dexDiffMd5);
                    } else {
                        logToDexMeta(newDexFile, oldDexFile, null, "0", relatedInfo.oldMd5, "0");
                    }
                }
            }
        }
```
由于md5不同时，前面的方法已经生成了classes.dex等差分文件，这里只需要把差异信息记录到dex_meta.txt中。当md5相同时，对于mainDex，需要移除掉loader classes之后，重新生成比对信息。
## 五、DexPatchGenerator
dex的差分工作主要在DexPatchGenerator中完成，
```java
public class DexPatchGenerator {
    private static final String TAG = "DexPatchGenerator";

    private final Dex oldDex;
    private final Dex newDex;
    private final DexPatcherLogger logger = new DexPatcherLogger();
    private DexSectionDiffAlgorithm<StringData> stringDataSectionDiffAlg;
    private DexSectionDiffAlgorithm<Integer> typeIdSectionDiffAlg;
    private DexSectionDiffAlgorithm<ProtoId> protoIdSectionDiffAlg;
    private DexSectionDiffAlgorithm<FieldId> fieldIdSectionDiffAlg;
    private DexSectionDiffAlgorithm<MethodId> methodIdSectionDiffAlg;
    private DexSectionDiffAlgorithm<ClassDef> classDefSectionDiffAlg;
    private DexSectionDiffAlgorithm<TypeList> typeListSectionDiffAlg;
    private DexSectionDiffAlgorithm<AnnotationSetRefList> annotationSetRefListSectionDiffAlg;
    private DexSectionDiffAlgorithm<AnnotationSet> annotationSetSectionDiffAlg;
    private DexSectionDiffAlgorithm<ClassData> classDataSectionDiffAlg;
    private DexSectionDiffAlgorithm<Code> codeSectionDiffAlg;
    private DexSectionDiffAlgorithm<DebugInfoItem> debugInfoSectionDiffAlg;
    private DexSectionDiffAlgorithm<Annotation> annotationSectionDiffAlg;
    private DexSectionDiffAlgorithm<EncodedValue> encodedArraySectionDiffAlg;
    private DexSectionDiffAlgorithm<AnnotationsDirectory> annotationsDirectorySectionDiffAlg;
    ......
}
```
### DexSectionDiffAlgorithm 
DexSectionDiffAlgorithm负责某一个section的差分工作，
```flow 
st=>start: 开始
e=>end: 结束
op1=>operation: 调整dex文件中元素的顺序
op2=>operation: 对比两个调整后dex的item
op3=>operation: patch操作的排序优化
op4=>operation: add和delete进行合并操作
op5=>operation: 记录patch过程中的操作
op6=>operation: 模拟patch过程

st->op1->op2->op3->op4->op5->op6->e
```
#### 调整dex文件中元素的顺序
在比较两个dex文件差异之前，需要按照固定的顺序去对dex中sectionItem进行排序
```java
        //按照一定的顺序调整  item的位置
        AbstractMap.SimpleEntry<Integer, T>[] adjustedOldIndexedItems = new AbstractMap.SimpleEntry[this.oldItemCount];
        System.arraycopy(this.adjustedOldIndexedItemsWithOrigOrder, 0, adjustedOldIndexedItems, 0, this.oldItemCount);
        Arrays.sort(adjustedOldIndexedItems, this.comparatorForItemDiff);

        AbstractMap.SimpleEntry<Integer, T>[] adjustedNewIndexedItems = collectSectionItems(this.newDex, false);
        this.newItemCount = adjustedNewIndexedItems.length;
        Arrays.sort(adjustedNewIndexedItems, this.comparatorForItemDiff);

```
其中有这么几个数据结构来保存对应关系，
```java
    /**
     * SparseIndexMap for mapping items between old dex and new dex.
     * e.g. item.oldIndex => item.newIndex
     */
    private final SparseIndexMap oldToNewIndexMap;
    /**
     * SparseIndexMap for mapping items between old dex and patched dex.
     * e.g. item.oldIndex => item.patchedIndex
     */
    private final SparseIndexMap oldToPatchedIndexMap;
    /**
     * SparseIndexMap for mapping items between new dex and patched dex.
     * e.g. item.newIndex => item.newIndexInPatchedDex
     */
    private final SparseIndexMap newToPatchedIndexMap;
        /**
     * SparseIndexMap for mapping items in new dex when skip items.
     */
    private final SparseIndexMap selfIndexMapForSkip;
```
**由于patchedDex和newDex不是完全相同，数据结构以及内容相同，但是排列顺序不一定相同，也就是patchedDex最终合成的dex和newDex的执行效果相同，但是md5不一定要相同，所以这里需要上面这么几个集合来保存oldDex,patchedDex和newDex的对应关系。**
调整item顺序之前中有个关键步骤，就是去收集一个dex文件中的item,这里实际就是**按照patched需要的顺序去调整oldDex和newDex中的数据结构**。
```java
 private AbstractMap.SimpleEntry<Integer, T>[] collectSectionItems(Dex dex, boolean isOldDex) {
        TableOfContents.Section tocSec = getTocSection(dex);
        if (!tocSec.exists()) {
            return EMPTY_ENTRY_ARRAY;
        }
        Dex.Section dexSec = dex.openSection(tocSec);
        int itemCount = tocSec.size;
        List<AbstractMap.SimpleEntry<Integer, T>> result = new ArrayList<>(itemCount);
        if (isOldDex) {
            for (int i = 0; i < itemCount; ++i) {
                T nextItem = nextItem(dexSec);
                T adjustedItem = adjustItem(oldToPatchedIndexMap, nextItem);   //按照patchedDex的顺序来调整
                result.add(new AbstractMap.SimpleEntry<>(i, adjustedItem)); //加入调整后的item
            }
        } else {
            int i = 0;
            while (i < itemCount) {
                T nextItem = nextItem(dexSec);
                int indexBeforeSkip = i;
                int offsetBeforeSkip = getItemOffsetOrIndex(indexBeforeSkip, nextItem);
                int indexAfterSkip = indexBeforeSkip;
                //跳过不需要  考虑的item
                while (indexAfterSkip < itemCount && shouldSkipInNewDex(nextItem)) {
                    if (indexAfterSkip + 1 >= itemCount) {
                        // after skipping last item, nextItem will be null.
                        nextItem = null;
                    } else {
                        nextItem = nextItem(dexSec);
                    }
                    ++indexAfterSkip;
                }
                if (nextItem != null) {
                    int offsetAfterSkip = getItemOffsetOrIndex(indexAfterSkip, nextItem);
                    //adjustItem(selfIndexMapForSkip, nextItem) 按照跳过的关系 拿到item，然后根据newToPatched的关系来调整newDex中item的顺序
                    T adjustedItem = adjustItem(newToPatchedIndexMap, adjustItem(selfIndexMapForSkip, nextItem));
                    int currentOutIndex = result.size();
                    result.add(new AbstractMap.SimpleEntry<>(currentOutIndex, adjustedItem));
                    updateIndexOrOffset(selfIndexMapForSkip, indexBeforeSkip, offsetBeforeSkip, indexAfterSkip, offsetAfterSkip);
                }
                i = indexAfterSkip;
                ++i;
            }
        }
        return result.toArray(new AbstractMap.SimpleEntry[0]);
    }
```
#### 对比两个调整后dex的item
根据比较的结果把操作的步骤放到patchOperationList中，
```java
    int oldCursor = 0;
        int newCursor = 0;
        while (oldCursor < this.oldItemCount || newCursor < this.newItemCount) {
            if (oldCursor >= this.oldItemCount) {
                // rest item are all newItem.
                //剩余的 全部是newItem内容
                while (newCursor < this.newItemCount) {
                    AbstractMap.SimpleEntry<Integer, T> newIndexedItem = adjustedNewIndexedItems[newCursor++];
                    this.patchOperationList.add(new PatchOperation<>(PatchOperation.OP_ADD, newIndexedItem.getKey(), newIndexedItem.getValue()));
                }
            } else
            if (newCursor >= newItemCount) {
                // rest item are all oldItem. 
                // 剩下的全是oldItem
                while (oldCursor < oldItemCount) {
                    AbstractMap.SimpleEntry<Integer, T> oldIndexedItem = adjustedOldIndexedItems[oldCursor++];
                    int deletedIndex = oldIndexedItem.getKey();
                    int deletedOffset = getItemOffsetOrIndex(deletedIndex, oldIndexedItem.getValue());
                    this.patchOperationList.add(new PatchOperation<T>(PatchOperation.OP_DEL, deletedIndex));
                    markDeletedIndexOrOffset(this.oldToPatchedIndexMap, deletedIndex, deletedOffset);
                }
            } else {
                //比较两者的差异
                AbstractMap.SimpleEntry<Integer, T> oldIndexedItem = adjustedOldIndexedItems[oldCursor];
                AbstractMap.SimpleEntry<Integer, T> newIndexedItem = adjustedNewIndexedItems[newCursor];
                int cmpRes = oldIndexedItem.getValue().compareTo(newIndexedItem.getValue());
                if (cmpRes < 0) {
                    //如果结果小于0  表示被删除
                    int deletedIndex = oldIndexedItem.getKey();
                    int deletedOffset = getItemOffsetOrIndex(deletedIndex, oldIndexedItem.getValue());
                    this.patchOperationList.add(new PatchOperation<T>(PatchOperation.OP_DEL, deletedIndex));
                    markDeletedIndexOrOffset(this.oldToPatchedIndexMap, deletedIndex, deletedOffset);
                    ++oldCursor;
                } else
                if (cmpRes > 0) {
                    //结果大于0  表示被增加
                    this.patchOperationList.add(new PatchOperation<>(PatchOperation.OP_ADD, newIndexedItem.getKey(), newIndexedItem.getValue()));
                    ++newCursor;
                } else {
                    //如果完全一样  只需要建立对应的偏移关系就可以了
                    int oldIndex = oldIndexedItem.getKey();
                    int newIndex = newIndexedItem.getKey();
                    int oldOffset = getItemOffsetOrIndex(oldIndexedItem.getKey(), oldIndexedItem.getValue());
                    int newOffset = getItemOffsetOrIndex(newIndexedItem.getKey(), newIndexedItem.getValue());

                    if (oldIndex != newIndex) {
                        this.oldIndexToNewIndexMap.put(oldIndex, newIndex);
                    }

                    if (oldOffset != newOffset) {
                        this.oldOffsetToNewOffsetMap.put(oldOffset, newOffset);
                    }

                    ++oldCursor;
                    ++newCursor;
                }
            }
        }

``` 

#### patch操作的排序优化
为了让patch操作更加优雅和简洁，需要按照一定顺序进行排序，主要是为了后面的合并步骤，
```java
        //对patch的操作进行排序优化
        Collections.sort(this.patchOperationList, comparatorForPatchOperationOpt);
```

#### add和delete进行合并操作
add和delete如果发生在同一个位置，可以合并为replace操作，
```java
  Iterator<PatchOperation<T>> patchOperationIt = this.patchOperationList.iterator();
        PatchOperation<T> prevPatchOperation = null;
        while (patchOperationIt.hasNext()) {
            PatchOperation<T> patchOperation = patchOperationIt.next();
            //对add和del进行replace合并操作
            if (prevPatchOperation != null
                && prevPatchOperation.op == PatchOperation.OP_DEL
                && patchOperation.op == PatchOperation.OP_ADD
            ) {
                if (prevPatchOperation.index == patchOperation.index) {
                    prevPatchOperation.op = PatchOperation.OP_REPLACE;
                    prevPatchOperation.newItem = patchOperation.newItem;
                    patchOperationIt.remove();
                    prevPatchOperation = null;
                } else {
                    prevPatchOperation = patchOperation;
                }
            } else {
                prevPatchOperation = patchOperation;
            }
        }
```

#### 记录patch过程中的操作
```java
 // Finally we record some information for the final calculations.
        patchOperationIt = this.patchOperationList.iterator();
        while (patchOperationIt.hasNext()) {
            PatchOperation<T> patchOperation = patchOperationIt.next();
            switch (patchOperation.op) {
                case PatchOperation.OP_DEL: {
                    indexToDelOperationMap.put(patchOperation.index, patchOperation);
                    break;
                }
                case PatchOperation.OP_ADD: {
                    indexToAddOperationMap.put(patchOperation.index, patchOperation);
                    break;
                }
                case PatchOperation.OP_REPLACE: {
                    indexToReplaceOperationMap.put(patchOperation.index, patchOperation);
                    break;
                }
            }
        }
```
#### 模拟patch过程
每次section进行上面的步骤后，都要模拟一次patch过程，主要为了计算patched之后的对应关系和section的size。
```java
    public void simulatePatchOperation(int baseOffset) {
        boolean isNeedToMakeAlign = getTocSection(this.oldDex).isElementFourByteAligned;
        int oldIndex = 0;
        int patchedIndex = 0;
        int patchedOffset = baseOffset;
        while (oldIndex < this.oldItemCount || patchedIndex < this.newItemCount) {
            if (this.indexToAddOperationMap.containsKey(patchedIndex)) {
                PatchOperation<T> patchOperation = this.indexToAddOperationMap.get(patchedIndex);
                if (isNeedToMakeAlign) {
                    patchedOffset = SizeOf.roundToTimesOfFour(patchedOffset);
                }
                T newItem = patchOperation.newItem;
                int itemSize = getItemSize(newItem);
                updateIndexOrOffset(
                        this.newToPatchedIndexMap,
                        0,
                        getItemOffsetOrIndex(patchOperation.index, newItem),
                        0,
                        patchedOffset
                );
                ++patchedIndex;
                patchedOffset += itemSize;
            } else
            if (this.indexToReplaceOperationMap.containsKey(patchedIndex)) {
                PatchOperation<T> patchOperation = this.indexToReplaceOperationMap.get(patchedIndex);
                if (isNeedToMakeAlign) {
                    patchedOffset = SizeOf.roundToTimesOfFour(patchedOffset);
                }
                T newItem = patchOperation.newItem;
                int itemSize = getItemSize(newItem);
                updateIndexOrOffset(
                        this.newToPatchedIndexMap,
                        0,
                        getItemOffsetOrIndex(patchOperation.index, newItem),
                        0,
                        patchedOffset
                );
                ++patchedIndex;
                patchedOffset += itemSize;
            } else
            if (this.indexToDelOperationMap.containsKey(oldIndex)) {
                ++oldIndex;
            } else
            if (this.indexToReplaceOperationMap.containsKey(oldIndex)) {
                ++oldIndex;
            } else
            if (oldIndex < this.oldItemCount) {
                if (isNeedToMakeAlign) {
                    patchedOffset = SizeOf.roundToTimesOfFour(patchedOffset);
                }

                T oldItem = this.adjustedOldIndexedItemsWithOrigOrder[oldIndex].getValue();
                int itemSize = getItemSize(oldItem);

                int oldOffset = getItemOffsetOrIndex(oldIndex, oldItem);

                updateIndexOrOffset(
                        this.oldToPatchedIndexMap,
                        oldIndex,
                        oldOffset,
                        patchedIndex,
                        patchedOffset
                );

                int newIndex = oldIndex;
                if (this.oldIndexToNewIndexMap.containsKey(oldIndex)) {
                    newIndex = this.oldIndexToNewIndexMap.get(oldIndex);
                }

                int newOffset = oldOffset;
                if (this.oldOffsetToNewOffsetMap.containsKey(oldOffset)) {
                    newOffset = this.oldOffsetToNewOffsetMap.get(oldOffset);
                }

                updateIndexOrOffset(
                        this.newToPatchedIndexMap,
                        newIndex,
                        newOffset,
                        patchedIndex,
                        patchedOffset
                );

                ++oldIndex;
                ++patchedIndex;
                patchedOffset += itemSize;
            }
        }

        this.patchedSectionSize = SizeOf.roundToTimesOfFour(patchedOffset - baseOffset);
    }
```

## 六、addTestDex
这里会加入test.dex，不进行详述。主要是能够更好地标识这一过程已经完成。

 


  [1]: http://on8vjlgub.bkt.clouddn.com/DexPatchDecoder.png "DexPatchDecoder"
  [2]: http://on8vjlgub.bkt.clouddn.com/collectAddedOrDeletedClasses.png
