---
title: OpenGL ES入门知识分享
date: 2017/11/10 13:44:00
author: 胡玮
tags:
- OpenGL ES 
categories: 
- OpenGL ES入门知识分享
---

本篇将介绍OpenGL ES基本知识以及相关实例，看完文章将会绘制简单几何图形，立方体，贴图，光影等等。
<!--more-->
## 一，OpenGL ES简介
OpenGL（Open Graphics Library）是个定义了一个跨编程语言、跨平台的编程接口的规范，它用于三维/二维图象。OpenGL是个专业的图形程序接口，是一个功能强大，调用方便的底层图形库。OpenGL ES (OpenGL for Embedded Systems) 是 OpenGL 三维图形 API 的子集，主要针对移动设备，目前主要包含1.x，2.0, 3.x等版本。

### 1.x版本
OpenGL ES1.X采用固定管线渲染，使用固定管线渲染时，不需要使用GLSL语言就行Shader程序的书写，在OpengGL环境配置方面简单许多。

### 2.0以及3.x版本
OpenGL ES2.0针对可编程管道，主要包含两部分：OpenGL ES2.0 API规范和OpenGL着色器语言。OpenGL负责把三维空间中的对象通过投影、光栅化转换为二维图像，然后呈现到屏幕上。OpenGL ES3.X版本是基于2.0版本的扩充与增强。

![enter description here][1]

## 二，顶点着色器
### １，输入
顶点着色器输入包括:
- 着色器程序　　　描述顶点上执行操作的顶点着色器程序源代码或者可执行文件
- 顶点着色器输入(或者属性)	　　用顶点数组提供的每个顶点的数据
- 统一变量　　　顶点（或者片段）着色器使用的不变数据
- 采样器	　　　代表顶点着色器使用纹理的特殊统一变量类型
顶点着色器的输出在2.0版本称作可变(varying)变量，用作输入传递给片段着色器。

### 2，内建变量
- gl_VertexID 是一个输入变量，用于保存顶点的整数索引。这个整数型变量用highp精度限定符申明。
- gl_InstanceID 是一个输入变量，用于保存实例化绘图中调用中图元的实例编号。
- gl_Position 用于输出顶点位置的剪裁坐标。用highp精度限定符限定。
- gl_PointSize 用于写入以像素表示的点精灵尺寸，在渲染点精灵时使用。浮点类型，用highp精度限定符限定。
- gl_FrontFacing 是一个特殊变量，但不是由顶点着色器直接写入的，而是根据顶点着色器生成的位置值和渲染的图元类型生成的。它是一个布尔变量。

### 3，图元装配
图元是三角形、直线或者点精灵等几何对象。图元的每个顶点被发送到顶点着色器的不同拷贝。在图元装配旗舰，这些顶点被合成图元。
对于每个图元，必须确定图元在视锥体区域内。如果没有完全在视锥体内，可能需要剪裁。完全处于视锥体外，它就会被抛弃。

### 4，光栅化
光栅化是把图元转换为一组二维片段的过程。

### 顶点着色器GLSL脚本示例
```glsl
attribute vec4 a_Position;

void main()
{
    gl_Position = a_Position;
    gl_PointSize = 10.0;
}
```

## 三，片段着色器
### 1，输入
-　着色器程序　　　描述片段上所执行的片段着色器程序源代码或者可执行文件
-　输入变量　	光栅化单元用插值为每个片段生成的顶点着色器输出
-　统一变量	顶点（或者片段）着色器使用的不变数据
-　采样器　　　　　代表片段着色器所用的纹理的特殊统一变量类型 　
-　代码		片段着色器源代码或者二进制代码，描述在片段上执行的操作

### 2,内建变量
- gl_FragCoord	片段着色器中的一个只读变量，这个变量保存片段的窗口相对坐标系(x,y,z,1/w)。
- gl_FrontFacing 片段着色器中的一个只读变量。true表示是正面，false表示是反面。默认是GL_CCW,逆时针方向是正面。
- gl_PointCoord 只读变量，在渲染点精灵时使用。
- gl_FragDepth 一个只写变量，覆盖片段的固定功能深度值。相关：深度测试

### 3,深度测试
深度缓冲区(Detph buffer)同颜色缓冲区(color buffer)是对应的，颜色缓冲区存储的像素的颜色信息，而深度缓冲区存储像素的深度信息。在决定是否绘制一个物体的表面时，首先将表面对应像素的深度值与当前深度缓冲区中的值进行比较，如果大于等于深度缓冲区中值，则丢弃这部分;否则利用这个像素对应的深度值和颜色值，分别更新深度缓冲区和颜色缓冲区。这一过程称之为深度测试(Depth Testing)。

解决问题：
在绘制3D场景的时候，我们需要决定哪些部分对观察者是可见的，或者说哪些部分对观察者不可见，对于不可见的部分，我们应该及早的丢弃。这种问题称之为隐藏面消除（Hidden surface elimination）,或者称之为找出可见面(Visible surface detemination)。

![enter description here][2]

![enter description here][3]

### 4,混合
一旦片段通过了所有启动的片段测试，它的颜色将与片段像素位置中已经存在的颜色组合。混合就是颜色混合的过程。混合方程如下：

```mathjax!
$$　C_\final = f_\source C_\source op f_\destionation C_\desination $$
```
 
### 5,抖动
在由于帧缓冲区中每个分量的位数导致的帧缓冲区中可用颜色数量有限的系统上，我们可以用抖动（Dither）模拟更大的色深。抖动算法以某种方式安排颜色，使图像看上去似乎比实际上可用的颜色更多。

### 片段着色器GLSL脚本示例
```glsl
precision mediump float;

uniform vec4 u_Color;

void main()
{
    gl_FragColor = u_Color;
}
```

## 四，着色器和程序
### 1，创建和编译一个着色器
获得链接后的着色器对象一般需要以下６个步骤，
１，创建一个顶点着色器对象和一个片段着色器对象。
２，将源代码连接到每个着色器对象。
３，编译着色器对象。
４，创建一个程序对象。
５，将编译后的着色器对象连接到程序对象。
６，链接程序对象。

具体函数：
- glCreateShader(GLenum type) 创建着色器
- glDeleteShader(GLunit shader)　删除着色器
- glShaderSource(GLuint shader, GLsizei count, const GLchar*, const *stirng. const GLint *length )
- glCompileShader(GLunit shader) 编译着色器
- glGetShaderiv(GLunit shader, GLemun pname, GLint *params) 

###　２，着色语言基础
#### 2.1，基本数据类型
![enter description here][4]

##### void
函数没有返回值必须声明为void，没有默认的函数返回值。关键字void不能用于其他声明，除了空形参列表外。

##### Booleans
布尔值，只有两个取值true或false。

##### Integers
整型主要作为编程的援助角色。在硬件级别上，真正地整数帮助有效的实现循环和数组索引，纹理单元索引。然而，着色语言没必要将整型数映射到硬件级别的整数。我们并不赞成底层硬件充分支持范围广泛的整数操作。OpenGL ES着色语言会把整数转化成浮点数进行操作。整型数可以使用十进制（非0开头数字），八进制（0开头数组）和十六进制表示（0x开头数字）。
**int i, j = 42;**

##### Float
浮点型用于广泛的标量计算。可以如下定义一个浮点数：
**float a, b = 1.5;**

##### Vectors
OpenGL ES着色语言包含像2-，3-， 4-浮点数、整数、booleans型向量的泛型表示法。浮点型向量可以保存各种有用的图形数据，如颜色，位置，纹理坐标。
**vec2 texCoord1, texCoord2;
vec3 position;
vec4 rgba;
ivec2 textureLookup;
bvec3 lessThan;**
向量的初始化工作可以放在构造函数中完成。

##### Matrices
矩阵是另一个在计算机图形中非常有用的数据类型，OpenGL ES着色语言支持2*2， 3*3, 4*4浮点数矩阵。
**mat2 mat2D;
mat3 optMatrix;
mat4 view, projection;**
矩阵的初始化工作可以放在构造函数中完成。

##### Sampler
采样器类型（如sampler2D）实际上是纹理的不透明句柄。它们用在内建的纹理函数来指明要访问哪一个纹理。它们只能被声明为函数参数或uniforms。除了纹理查找函数参数， 数组索引， 结构体字段选择和圆括号外，取样器不允许出现在表达式中。取样器不能作为左值。这些限制同样适用于包含取样器的的结构体。作为uniforms时，它们通过OpenGL ES API初始化。作为函数参数，仅能传入匹配的采样器类型。这样可以在着色器运行之前进行着色器纹理访问和OpenGL ES纹理状态的一致性检查。

#### 2.2 uniform变量
OpenGL ES着色语言中的变量类型限定符之一是统一变量。统一变量存储应用程序通过OpenGL ES 3.0 API传入着色器的只读值，对于保存着色器所需的所有数据类型（比如变换矩阵、照明参数和颜色）都很有用。统一变量用uniform声明，如下：
**uniform mat4 viewProjMatrix;
uniform mat4 viewMatrix;
uniform vec3 lightPosition;**
统一变量的命名空间在顶点着色器和片段着色器中是共享的。也就是说，如果顶点着色器和片段着色器链接到一个程序对象，他们就会共享同一组统一变量。因此，如果顶点和片段着色器中都声明一个统一变量，那么这两个声明必须匹配。

#### 2.3 attribute变量
attribute变量是只能在vertex shader中使用的变量。（它不能在fragment shader中声明attribute变量，也不能被fragment shader中使用）

一般用attribute变量来表示一些顶点的数据，如：顶点坐标，法线，纹理坐标，顶点颜色等。

在application中，一般用函数glBindAttribLocation（）来绑定每个attribute变量的位置，然后用函数glVertexAttribPointer（）为每个attribute变量赋值。

以下是例子：
```glsl
uniform mat4 u_matViewProjection;
attribute vec4 a_position;
attribute vec2 a_texCoord0;
varying vec2 v_texCoord;
void main(void)
{
gl_Position = u_matViewProjection * a_position;
v_texCoord = a_texCoord0;
}
```

#### 2.4 varying变量
varying变量是vertex和fragment shader之间做数据传递用的。一般vertex shader修改varying变量的值，然后fragment shader使用该varying变量的值。因此varying变量在vertex和fragment shader二者之间的声明必须是一致的。application不能使用此变量。

#### 2.5 精度限定符
精度限定符可以用于制定任何基于浮点数或者整数变量的精度。指定精度的关键字是lowp、mediump和highp。用如下语法指定：
precision highp float;
precision mediump int;

在没有指定精度也限定符声明的便令都默认使用最高精度highp。

## 五，实例
![enter description here][5]

![enter description here][6]

![enter description here][7]

![enter description here][8]


  [1]: http://on8vjlgub.bkt.clouddn.com/1.jpg "1"
  [2]: http://on8vjlgub.bkt.clouddn.com/20160807203914290.png "绘制顺序"
  [3]: http://on8vjlgub.bkt.clouddn.com/20160807203943775.png "特殊情况"
  [4]: http://on8vjlgub.bkt.clouddn.com/basetype.jpg "基本数据类型"
  [5]: http://on8vjlgub.bkt.clouddn.com/device-2017-10-30-102229.png "三角形"
  [6]: http://on8vjlgub.bkt.clouddn.com/device-2017-10-30-102246.png "矩形"
  [7]: http://on8vjlgub.bkt.clouddn.com/device-2017-10-30-102316.png "立方体"
  [8]: http://on8vjlgub.bkt.clouddn.com/device-2017-10-30-102334.png "带贴图的立方体"
