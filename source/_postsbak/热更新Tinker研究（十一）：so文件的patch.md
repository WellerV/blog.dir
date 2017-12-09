---

title: 热更新Tinker研究（十一）：so文件的patch
date: 2017/04/20 14:34:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

## 一、重要文件说明
### 1,关于so_meta.txt
相邻元素用‘，’分隔，不同的item用行分隔。
示例如下：
```
libHelloJNI.so,lib/x86,8bee6a211b80de30bd20a3c84f10c606,1094172186,2b9e5ecfe8ee8ad0f4cd72bce783e662
libHelloJNI.so,lib/armeabi-v7a,bfa1eeb03f9b5d0c8bfa46b30384b436,2738641823,e8dcb4455b0b3395a29599f10def7ccc
libHelloJNI.so,lib/armeabi,6f7065d883f7e9687967164e08f1913a,3024247562,6a7492ed0923f006537797d8df547d60
```
具体如下面的表格
| name  | path | md5 | rawCrc | patchMd5 |
| -------  | ----- | ------------------  | ----------------- | --------------- | ------------- | 
|libHelloJNI.so	|lib/x86	|8bee6a211b80de30bd20a3c84f10c606	|1094172186	|2b9e5ecfe8ee8ad0f4cd72bce783e662
|libHelloJNI.so| lib/armeabi-v7a	|bfa1eeb03f9b5d0c8bfa46b30384b436	|2738641823	|e8dcb4455b0b3395a29599f10def7ccc
|libHelloJNI.so	|lib/armeabi	|6f7065d883f7e9687967164e08f1913a 	|3024247562	|6a7492ed0923f006537797d8df547d60

name表示so文件的名称，path表示路径，md5是新生成so文件的校验值，rawCrc是baskApk的crc,patchMd5是patch.so对应的md5值。

### 2,patch.so的格式
![enter description here][1]

magic为标识，这里没有进行特殊的校验。ctrlBlockLen为第二个区域控制区域的长度，diffBlockLen为diffBlockData的长度，extraBlockLen为extraBlockData的长度。ctrl[]的三个值依次代表diffBlockData区域需要读取的长度，extraBlockData区域需要读取的长度以及oldPos的偏移量。


## 二、BSPatch中的patch
![enter description here][2]
这里根据newPos循环来控制patch的过程，
```java

   // byte[] newBuf = new byte[newsize + 1];
        byte[] newBuf = new byte[newsize];

        int oldpos = 0;
        int newpos = 0;
        int[] ctrl = new int[3];    //控制数组

        // int nbytes;
        while (newpos < newsize) {

            for (int i = 0; i <= 2; i++) {
                ctrl[i] = ctrlBlockIn.readInt();
            }

            // ctrl[0]
            if (newpos + ctrl[0] > newsize) {
                throw new IOException("Corrupt by wrong patch file.");
            }

            // Read ctrl[0] bytes from diffBlock stream
            if (!BSUtil.readFromStream(diffBlockIn, newBuf, newpos, ctrl[0])) {
                throw new IOException("Corrupt by wrong patch file.");
            }

            for (int i = 0; i < ctrl[0]; i++) {
                if ((oldpos + i >= 0) && (oldpos + i < oldsize)) {
                    newBuf[newpos + i] += oldBuf[oldpos + i];
                }
            }

            newpos += ctrl[0];
            oldpos += ctrl[0];

            if (newpos + ctrl[1] > newsize) {
                throw new IOException("Corrupt by wrong patch file.");
            }

            if (!BSUtil.readFromStream(extraBlockIn, newBuf, newpos, ctrl[1])) {
                throw new IOException("Corrupt by wrong patch file.");
            }

            newpos += ctrl[1];
            oldpos += ctrl[2];
        }
        ctrlBlockIn.close();
        diffBlockIn.close();
        extraBlockIn.close();

        return newBuf;

```
首先是读取ctrl[]数组,然后从diffBuf中读取ctrl[0]长度的数据到newBuf，再加上oldBuf中对应位置的数据，得到蓝色区域。后面再是从extraBuf中读取长度为ctrl[1]的数据，最后再将oldPos偏移ctrl[2],重复上面的循环。最终得到新的so文件。


  [1]: http://on8vjlgub.bkt.clouddn.com/BSPatchFile.png "BSPatchFile"
  [2]: http://on8vjlgub.bkt.clouddn.com/BSPatch.png "BSPatch"
