---

title: 热更新Tinker研究（二）：结合源码学习Dex格式
date: 2017/03/15 15:56:00
tags:
- Android 
- 补丁
- 热更新
- 源码分析
categories: 
- 基于源码的热更新Tinker框架研究

---

## 一、Dex文件中的数据类型
Name	| 	Description
----		|	------	|
byte		| 	1字节有符号数
ubyte	|	1字节无符号数
short	|	2字节有符号数, 低字节序
ushort	|	2字节无符号数, 低字节序
int		|	4字节有符号数, 低字节序
uint		|	4字节无符号数, 低字节序
long		|	8字节有符号数, 低字节序
ulong	|	8字节无符号数, 低字节序
sleb128	|	有符号 LEB128，可变长度 1~5 字节
uleb128	|	无符号 LEB128，可变长度 1~5 字节
uleb128p1	|	无符号 LEB128 值加1，可变长 1~5 字节

### LEB128
这种编码是变长的，诞生于DWARF3 标准。在dex中,LEB128只能编码成32位的数字，而长度是1-5字节。每个字节上保留第一位，剩余7位是有效位。如下图。

![这里写图片描述](http://img.blog.csdn.net/20170315105049472?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由于采用低字节序，第一个字节表示低位。

uleb128 存储方式类似，只不过表示无符号数字。uleb128p表示在此基础上+1。

![这里写图片描述](http://img.blog.csdn.net/20170315105923228?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 二、Dex数据结构
结合这张图来分析
![这里写图片描述](http://img.blog.csdn.net/20151113110209637?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
整个dex文件可以分为三大部分，

- header头文件
- 索引区
- 数据区

**header头文件**主要包含Dex Header，主要是dex文件中整个区块的索引和偏移，以确定各个区块在什么位置。**索引区**ids是identifiers的缩写，主要指向数据区的位置和偏移。**数据区**主要是存放类结构的定义，包含权限、类id、父类、字段、方法等。

以下是一段HelloWorld代码
```java
public class HelloWorld{
    public static void main(String[] args){
        System.out.println("HelloWorld");
    }
}
```
生成的HelloWorld.dex文件在[010 Editor](http://www.sweetscape.com/010editor/)中的显示
![这里写图片描述](http://img.blog.csdn.net/20170315111916588?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### header
![这里写图片描述](http://img.blog.csdn.net/20170315112457753?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
#### magic
固定值，用来标识dex文件，值为dex 035，
二进制表示
```
{0x64, 0x65, 0x78, 0x0A, 0x30, 0x33, 0x35, 0x00} = "dex\n035\0"
```
#### checksum
文件校验码，使用 alder32 算法校验文件除去 maigc、checksum 外余下的所有文件区域，用于检查文件错误。
#### signature
使用 SHA-1 算法 hash 除去 magic、checksum 和 signature 外余下的所有文件区域， 用于唯一识别本文件 。
#### file_size
文件大小
#### endian_tag
字节顺序,其中12345678h表示低字节序，78563412h表示高字节序。
#### 其他
其他内容分别表示各个数据区域的偏移。
### string_ids
![这里写图片描述](http://img.blog.csdn.net/20170315113803271?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
其中string_id_item下,
#### string_data_off
string数据的偏移
#### struct uleb128 utf16_size
UTF-16编码下字符串长度
#### string data[]
MUTF-8编码构成的字符串，MUTF-8（Modified UTF-8）其实是对UTF-16字符编码的再编码。
### type_ids
![这里写图片描述](http://img.blog.csdn.net/20170315114653236?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
#### uint descriptor_idx
uint类型的id,0x3表示在string_ids[]中的游标
### proto_ids
![这里写图片描述](http://img.blog.csdn.net/20170315135321067?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
proto 的意思是 method prototype 代表 java 语言里的一个 method 的原型 
#### uint shorty_idx
简短的方法描述符，它的值是string_ids的index。
#### uint return_type_idx
返回类型,它的值是type_ids的id。
#### uint parameters_off
参数列表的偏移
#### struct type_item_list parameters
参数列表,其中uint size表示参数个数,struct type_item list[]表示具体的参数列表。
### field_ids
![这里写图片描述](http://img.blog.csdn.net/20170315140612600?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
#### ushort class_idx
表示字段属于的class类，它的值是class_def的index。
#### ushort type_idx
表示该字段的类型，它的值是type_ids的index。
#### uint name_idx
表示字段名称，它的值是string_ids的index。
### method_ids
 ![这里写图片描述](http://img.blog.csdn.net/20170315142217197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### ushort class_idx
表示方法属于的class类,它的值是class_def的id。由于这里是ushort类型，所以最大的方法数不能超过2^16个。
#### ushort proto_idx
表示方法的类型，它的值proto_ids的index。
#### uint name_idx
表示方法的名称，它的值string_ids的index。
### class_defs
![这里写图片描述](http://img.blog.csdn.net/20170315143650893?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
由于这个结构比较复杂，主要包括class_def_item和class_data_item，下面分别来分析。
### class_def_item
#### class_idx
描述具体的 class 类型，值是 type_ids 的一个 index 。值必须是一个 class 类型，不能是数组类型或者基本类型。
#### access_flags
描述 class 的访问类型，比如 public, enum, final, static等。[access_flags definitions](http://source.android.com/devices/tech/dalvik/dex-format.html)会有描述。
#### superclass_idx
描述 supperclass 的类型，值的形式跟 class_idx 一样 。
#### interfaces_off
值为偏移地址，指向 class 的 interfaces，被指向的数据结构为 type_list 。class 若没有 interfaces 值为 0。
#### source_file_idx
表示源代码文件的信息，值是 string_ids 的一个 index。若此项信息缺失，此项值赋值为 NO_INDEX=0xffff ffff。
#### annotions_off
值是一个偏移地址，指向的内容是该 class 的注释，位置在 data 区，格式为 annotations_direcotry_item。若没有此项内容，值为 0 。
#### class_data_off
值是一个偏移地址，指向的内容是该 class 的使用到的数据，位置在 data 区，格式为 class_data_item。若没有此项内容值为 0。该结构里有很多内容，详细描述该 class 的 field、method, method 里的执行代码等信息，后面会介绍class_data_item
#### static_value_off
值是一个偏移地址 ，指向 data 区里的一个列表 (list)，格式为 encoded_array_item。若没有此项内容值为 0。
### class_data_item
#### struct uleb128 static_fields_size
描述静态方法数目
#### struct uleb128 instance_fields_size
描述实例方法数目
#### struct uleb128 direct_methods_size
描述直接方法数目
#### struct uleb128 virtual_methods_size
描述虚方法数目
#### struct encoded_method_list direct_methods
描述已有方法列表
### code_item
描述方法的详细构成(省略)
### map_list
![这里写图片描述](http://img.blog.csdn.net/20170315150450759?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
map_list内容和header类似，包含各个区域的大小和偏移，但是内容更为丰富和详细，
![这里写图片描述](http://img.blog.csdn.net/20170315150732877?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHV3ZWlnb29kYm95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
## 三、结合tinker源码
### com.tencent.tinker.android.dex.Dex
dex数据结构的抽象
```java
public final class Dex {
    ...
    private static final int CHECKSUM_OFFSET = 8;   //checksum偏移位置
    private static final int SIGNATURE_OFFSET = CHECKSUM_OFFSET + SizeOf.CHECKSUM;  //signature偏移位置
    private final TableOfContents tableOfContents = new TableOfContents();  //对应header 和 map
    private final StringTable strings = new StringTable();  //对应string_ids
    private final TypeIndexToDescriptorIndexTable typeIds = new TypeIndexToDescriptorIndexTable();  //对应type_ids
    private final TypeIndexToDescriptorTable typeNames = new TypeIndexToDescriptorTable();  //抽象出type_name的数据结构
    private final ProtoIdTable protoIds = new ProtoIdTable();   //对应proto_ids
    private final FieldIdTable fieldIds = new FieldIdTable();   //对应field_ids
    private final MethodIdTable methodIds = new MethodIdTable();    //对应method_ids
    private final ClassDefTable classDefs = new ClassDefTable();    //对应class_defs
    ...
}
```

### com.tencent.tinker.android.dex.TableOfContents
描述header和map
```java
public final class TableOfContents {
    ...
    public static final short SECTION_TYPE_HEADER = 0x0000;
    public static final short SECTION_TYPE_STRINGIDS = 0x0001;
    public static final short SECTION_TYPE_TYPEIDS = 0x0002;
    public static final short SECTION_TYPE_PROTOIDS = 0x0003;
    public static final short SECTION_TYPE_FIELDIDS = 0x0004;
    public static final short SECTION_TYPE_METHODIDS = 0x0005;
    public static final short SECTION_TYPE_CLASSDEFS = 0x0006;
    public static final short SECTION_TYPE_MAPLIST = 0x1000;
    public static final short SECTION_TYPE_TYPELISTS = 0x1001;
    public static final short SECTION_TYPE_ANNOTATIONSETREFLISTS = 0x1002;
    public static final short SECTION_TYPE_ANNOTATIONSETS = 0x1003;
    public static final short SECTION_TYPE_CLASSDATA = 0x2000;
    public static final short SECTION_TYPE_CODES = 0x2001;
    public static final short SECTION_TYPE_STRINGDATAS = 0x2002;
    public static final short SECTION_TYPE_DEBUGINFOS = 0x2003;
    public static final short SECTION_TYPE_ANNOTATIONS = 0x2004;
    public static final short SECTION_TYPE_ENCODEDARRAYS = 0x2005;
    public static final short SECTION_TYPE_ANNOTATIONSDIRECTORIES = 0x2006;

    //各类Section 索引区
    public final Section header = new Section(SECTION_TYPE_HEADER, true);   
    public final Section stringIds = new Section(SECTION_TYPE_STRINGIDS, true);
    public final Section typeIds = new Section(SECTION_TYPE_TYPEIDS, true);
    public final Section protoIds = new Section(SECTION_TYPE_PROTOIDS, true);
    public final Section fieldIds = new Section(SECTION_TYPE_FIELDIDS, true);
    public final Section methodIds = new Section(SECTION_TYPE_METHODIDS, true);
    public final Section classDefs = new Section(SECTION_TYPE_CLASSDEFS, true);
    public final Section mapList = new Section(SECTION_TYPE_MAPLIST, true);
    public final Section typeLists = new Section(SECTION_TYPE_TYPELISTS, true);
    public final Section annotationSetRefLists = new Section(SECTION_TYPE_ANNOTATIONSETREFLISTS, true);
    public final Section annotationSets = new Section(SECTION_TYPE_ANNOTATIONSETS, true);
    public final Section classDatas = new Section(SECTION_TYPE_CLASSDATA, false);
    public final Section codes = new Section(SECTION_TYPE_CODES, true);
    public final Section stringDatas = new Section(SECTION_TYPE_STRINGDATAS, false);
    public final Section debugInfos = new Section(SECTION_TYPE_DEBUGINFOS, false);
    public final Section annotations = new Section(SECTION_TYPE_ANNOTATIONS, false);
    public final Section encodedArrays = new Section(SECTION_TYPE_ENCODEDARRAYS, false);
    public final Section annotationsDirectories = new Section(SECTION_TYPE_ANNOTATIONSDIRECTORIES, true);
    public final Section[] sections = {
            header, stringIds, typeIds, protoIds, fieldIds, methodIds, classDefs, mapList,
            typeLists, annotationSetRefLists, annotationSets, classDatas, codes, stringDatas,
            debugInfos, annotations, encodedArrays, annotationsDirectories
    };

    public int checksum;    //checksum校验值
    public byte[] signature;    //signatur识别签名
    public int fileSize;    
    public int linkSize;
    public int linkOff;
    public int dataSize;
    public int dataOff;
    ...
}
```

### com.tencent.tinker.android.dex.TableOfContents.Section
描述选区大小和偏移量
```java
public static class Section implements Comparable<Section> {
        public static final int UNDEF_INDEX = -1;
        public static final int UNDEF_OFFSET = -1;
        public final short type;
        public boolean isElementFourByteAligned;
        public int size = 0;
        public int off = UNDEF_OFFSET;
        public int byteCount = 0;
        ...
 }
```
### com.tencent.tinker.android.dex.Dex.StringTable
string表描述
```java
    private final class StringTable extends AbstractList<String> implements RandomAccess {
        @Override public String get(int index) {
            checkBounds(index, tableOfContents.stringIds.size);
            int stringOff = openSection(tableOfContents.stringIds.off + (index * SizeOf.STRING_ID_ITEM)).readInt();
            return openSection(stringOff).readStringData().value;
        }
        @Override public int size() {
            return tableOfContents.stringIds.size;
        }
    }
```
整个数据结构是基于Selection进行构建，只需要保存一些大小和偏移，需要取值时操作data即可，data也就是dex文件的二进制数组。