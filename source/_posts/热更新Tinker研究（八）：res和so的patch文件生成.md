---

title:  热更新Tinker研究（八）：res和so的patch文件生成
date: 2017/04/20 14:25:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---


ResDiffDecoder和BsDiffDecoder分别是负责resource和so文件的patch生成相关的，它们很多地方比较相似，这里放在一起来说明。


## 一、ResDiffDecoder
ResDiffDecoder是控制resources的patch文件生成的，主要是控制增加、修改和删除的信息，这里对于大文件和小文件也有不同的区分，小文件只需要直接拷贝，大文件需要做差分文件。
```java
public class ResDiffDecoder extends BaseDecoder {
    private static final String TEST_RESOURCE_NAME        = "only_use_to_test_tinker_resource.txt";
    private static final String TEST_RESOURCE_ASSETS_PATH = "assets/" + TEST_RESOURCE_NAME;

    private static final String TEMP_RES_ZIP  = "temp_res.zip";
    private static final String TEMP_RES_7ZIP = "temp_res_7ZIP.zip";
    private final InfoWriter                     logWriter;
    private final InfoWriter                     metaWriter;
    private       ArrayList<String>              addedSet;  //增加的
    private       ArrayList<String>              modifiedSet;   //修改的
    private       ArrayList<String>              largeModifiedSet;  ///修改的大文件
    private       HashMap<String, LargeModeInfo> largeModifiedMap;  //修改的大文件map保存
    private ArrayList<String> deletedSet;   //删除的
    ......
}
```
dealWithModeFile()是主要进行区分大小文件处理的方法，
```java
    private boolean dealWithModeFile(String name, String newMd5, File oldFile, File newFile, File outputFile) throws IOException {
        if (checkLargeModFile(newFile)) {   //判断是否是大文件
            if (!outputFile.getParentFile().exists()) { //确保父目录存在
                outputFile.getParentFile().mkdirs();
            }
            BSDiff.bsdiff(oldFile, newFile, outputFile);    //生成diff文件
            //treat it as normal modify
            //检查是否需要当做diff文件对待
            if (Utils.checkBsDiffFileSize(outputFile, newFile)) {
                LargeModeInfo largeModeInfo = new LargeModeInfo();
                largeModeInfo.path = newFile;
                largeModeInfo.crc = FileOperation.getFileCrc32(newFile);
                largeModeInfo.md5 = newMd5;
                largeModifiedSet.add(name); //加入large信息
                largeModifiedMap.put(name, largeModeInfo);
                writeResLog(newFile, oldFile, TypedValue.LARGE_MOD);
                return true;
            }
        }
        modifiedSet.add(name);
        //将新的文件拷贝目标路径
        FileOperation.copyFileUsingStream(newFile, outputFile);
        writeResLog(newFile, oldFile, TypedValue.MOD);
        return false;
    }
```
这里会判断新的文件是否是大文件，如果是大文件，就先用BSDiff生成diff文件，如果判断diff文件符合要求，就按照大文件进行处理。
对于大文件的判断，
```java
    private boolean checkLargeModFile(File file) {
        long length = file.length();
        if (length > config.mLargeModSize * TypedValue.K_BYTES) {
            return true;
        }
        return false;
    }

```
对于是否需要按照大文件处理，
```java
    public static boolean checkBsDiffFileSize(File bsDiffFile, File newFile) {
        if (!bsDiffFile.exists()) {
            throw new TinkerPatchException("can not find the bsDiff file:" + bsDiffFile.getAbsolutePath());
        }

        //check bsDiffFile file size
        double ratio = bsDiffFile.length() / (double) newFile.length();
        if (ratio > TypedValue.BSDIFF_PATCH_MAX_RATIO) {
            //如果diff比newFile的0.8倍还大，按照普通文件处理
            Logger.e("bsDiff patch file:%s, size:%dk, new file:%s, size:%dk. patch file is too large, treat it as newly file to save patch time!",
                bsDiffFile.getName(),
                bsDiffFile.length() / 1024,
                newFile.getName(),
                newFile.length() / 1024
            );
            return false;
        }
        return true;
    }
```

## 二、BsDiffDecoder
BsDiffDecoder是负责os文件差分的，区别是没有resources那种大小文件的区分，关键性代码如下，
```java
 public boolean patch(File oldFile, File newFile) throws IOException, TinkerPatchException {
        //first of all, we should check input files
        if (newFile == null || !newFile.exists()) {
            return false;
        }
        //new add file
        String newMd5 = MD5.getMD5(newFile);
        File bsDiffFile = getOutputPath(newFile).toFile();

        if (oldFile == null || !oldFile.exists()) {
            FileOperation.copyFileUsingStream(newFile, bsDiffFile);
            writeLogFiles(newFile, null, null, newMd5);
            return true;
        }

        //both file length is 0
        if (oldFile.length() == 0 && newFile.length() == 0) {
            return false;
        }
        if (oldFile.length() == 0 || newFile.length() == 0) {
            FileOperation.copyFileUsingStream(newFile, bsDiffFile);
            writeLogFiles(newFile, null, null, newMd5);
            return true;
        }

        //new add file
        String oldMd5 = MD5.getMD5(oldFile);

        if (oldMd5.equals(newMd5)) {
            return false;
        }

        if (!bsDiffFile.getParentFile().exists()) {
            bsDiffFile.getParentFile().mkdirs();
        }
        BSDiff.bsdiff(oldFile, newFile, bsDiffFile);

        if (Utils.checkBsDiffFileSize(bsDiffFile, newFile)) {
            writeLogFiles(newFile, oldFile, bsDiffFile, newMd5);
        } else {
            FileOperation.copyFileUsingStream(newFile, bsDiffFile);
            writeLogFiles(newFile, null, null, newMd5);
        }
        return true;
    }
```

## 三、bsdiff算法
bsdiff算法的核心思想是构造两个数组，找出公共序列的位置，一个是diff数组,一个是extra数组。diff数组是两个序列之间的差，extra表示new多余old的部分

求公共子串，需要用到后缀数组，这里关于关于后缀数组，做几个基本的名词的解释。

**字符串的大小比较：** 关于字符串的大小比较，是指通常所说的 “ 字典顺序 ” 比较， 也就是对于两个字符串 u 、v ，令 i 从 1 开始顺次比较 u[i] 和 v[i] ，如果u[i]=v[i] 则令 i 加 1 ，否则若 u[i] < v[i] 则认为 u < v ，u[i] > v[i] 则认为 u > v，比较结束。如果 i > len(u) 或者 i > len(v) 仍比较不出结果，那么若 len(u) < len(v)则认为 u < v ， 若 len(u)=len(v) 则 认 为 u=v ，若 len(u) > len(v) 则 u > v 。
 注：从字符串的大小比较的定义看，字符串s的所有后缀中任其中一对(u,v)不可能会相等，因为必要条件 len(u) ≠ len(v)不可能满足。所以任一字符串s中有len(s)个互不相同的后缀。我们可以将s的所有后缀排列，利用 后缀数组sa 与 名次数组rank 储存。 
 
**后缀数组sa：** 将s的n个后缀从小到大排序后将 排序后的后缀的开头位置 顺次放入sa中，则sa[i]储存的是排第i大的后缀的开头位置。简单的记忆就是“排第几的是谁”。

**名次数组rank：** rank[i]保存的是suffix(i)｛后缀｝在所有后缀中从小到大排列的名次。则 若 sa[i]=j，则 rank[j]=i。简单的记忆就是“你排第几”。

![enter description here][1]

![enter description here][2]

然后再根据公共子串的位置去得到diff和extra数组。这个过程随后会补图说明。


  [1]: http://on8vjlgub.bkt.clouddn.com/20120630fc_suffix.png "后缀"
  [2]: http://on8vjlgub.bkt.clouddn.com/400px-20120630fc_sa&rank.png "后缀数组和排名数组"
