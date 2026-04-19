---
title: shader介绍
date: 2023-05-14 17:48:35
description: "shader介绍"
tags: "shader"
categories: "shader"
cover: /covers/25.webp
---

## shader基本用法总结

<!-- more -->

### 1. Shader 基本介绍 

首先需要认识到的一点是，shader是运行在GPU上的，天然的**并行处理**

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.4ooooia27cs0.webp)

我们通过编写代码(即shader），**来‘点亮’下面画布中的每个像素**！

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.6mwney17yqk0.webp)

右边列出这么长的公式不是为了劝退大家，它都是由基本图形公式组合而来（后面我会给大家进行讲解），理解了就容易，但是一下子放在这里就很难理解，所以大家在写shader复合效果代码时，**一定要写注释说明**，不然后面优化维护的同学是真的难受😭

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // create 100x100 cell sheet
    float c = floor(100.0*(0.5+(fragCoord.x-0.5*iResolution.x)/iResolution.y));
    float r = floor(100.0*(1.0-fragCoord.y/iResolution.y));

    // paint flower
    float f = min(max(20.0+20.0*(pow(0.5+0.5*cos(5.0*atan(r-50.5,c-50.5)),0.3))-sqrt((c-50.5)*(c-50.5)+(r-50.5)*(r-50.5)),0.0),1.0);
    f += min(max(19.0-sqrt((c-50.5)*(c-50.5)+(r-50.5)*(r-50.5)),0.0),1.0);
    f -= 2.0*min(max(46.0-r,0.0),1.0)*min(max(2.0-abs(4.0-sqrt( pow(abs(r-45.0),2.0)+pow(abs(abs(c-50.5)-6.0),2.0))),0.0),1.0);
    f -= 2.0*min(max(r-50.  
    
    
    0,0.0),1.0)*min(max(2.0-abs(8.0-sqrt((c-50.5)*(c-50.5)+(r-50.5)*(r-50.5))),0.0),1.0);
    
    // colorize
    vec3 h = (f<1.0) ? mix(vec3(0.44,0.66,0.86),vec3(1),f) :
                       mix(vec3(1), vec3(1,0.85,0.4),f-1.0);

    // output
    fragColor = vec4(h,1.0);
}
```

#### 1.1 how to write a shader ?

- **我们在哪儿作画？**
  - https://www.shadertoy.com/new
  - VS Code 插件shadertoy, 使用方法也比较简单
- **一些基本知识介绍**

- 从哪儿开始呢？画一个三角形貌似有点难(PS：图形学中的hello world就是绘制一个三角形🔼)，那就从点亮屏幕开始吧 ！

```glsl
// shader具有一些内置函数：
//     iResolution :  画布的精度，宽高参数
//     iTime       :  时间参数，让效果动起来
//     iMouse      :  鼠标操作，交互设计
//     工具函数     :  sin, cos, pow, mod, smoothstep, floor等
 void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
     // 将画布归一化到0.0到1.0之间
    vec2 uv = fragCoord.xy / iResolution.xy;   

    //           输出颜色，即对应uv下的颜色值
    //                r     g    b   a
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
    
    // 对应图2
    // fragColor = vec4(uv.x, 0.0, 0.0, 1.0);
    
    // 对应图3
    // vec3 color = 0.5 + 0.5*cos(iTime+uv.xyx+vec3(0,2,4)); 
    // fragColor = vec4(color, 1.0);
}      
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.3yl1khhw3b00.webp" alt="image" style="zoom:80%;" />![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.6d2rqxyrgq00.webp)<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/c0afb7c7-ad91-427b-878a-d7967a5d15d3.2dxd0t8qnz40.gif" alt="c0afb7c7-ad91-427b-878a-d7967a5d15d3" style="zoom:40%;" />

​                    图一                                                         图二                                                             图三

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.19yzzfokpbr4.webp)

#### 1.2 generative art

##### 1.2.1 形状

- **善于使用step 和smoothstep函数**

​     [ step 函数](https://thebookofshaders.com/glossary/?search=step)：一般用来作截断处理

​      [smoothstep函数](https://thebookofshaders.com/glossary/?search=smoothstep)：一般用来做平滑处理，smoothstep函数可以替代step函数（参数1等于参数2，或者接近时，可以理解为step函数）

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord/iResolution.xy;
    uv = uv - 0.5;
     
    vec3 color = vec3(0.7, 0.4, 0.6);
    if (uv.x < -.3 && uv.y >0.1)
        color = vec3(0.7, 0.0, .0);
    if (uv.x > 0.2 && uv.y < -0.4)
        color = vec3(0.0, 0.0, 0.5);
    if (uv.x > 0.4 && uv.y > 0.1)
        color = vec3(0.0, 0.5, 0.6);
        
    float r = 0.01;
    color *= step(r, abs(uv.x + 0.3));
    color *= step(r, abs(uv.x - 0.2));
    color *= step(r, abs(uv.x - 0.4));
    color *= 1.0 -(1.0 - step(r, abs(uv.x + 0.4))) * (smoothstep(0.1, 0.1, uv.y));
    color *= step(r * 2., abs(uv.y - 0.1));
    color *= step(r * 2., abs(uv.y - 0.3));
    color *= 1.0 - (1.0 - step(r * 2., abs(uv.y + 0.4))) * smoothstep(-0.3, -0.3, uv.x);     
   
    fragColor = vec4(color,1.0);
}
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.5hxd4298hjg0.webp" alt="image" style="zoom:80%;" />                                     ![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.68rou7uosh40.webp)  

​              参考示意图                                                                                            运行代码图

- **SDF(符号距离场）**

> 详细可参考文档[Signed Distance Field](https://quvideo.feishu.cn/docs/doccnhRzxWM8QLgecBYUCFik30c) 

在这一部分，使用最频繁的函数是 length(), 其它具有同样功能的有sqrt(),distance()

这里主要是掌握距离场这种思想！！！

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    // 将坐标原点移到中心
    uv = uv - 0.5;
    // 保持原来的长宽比
    uv.x *= iResolution.x / iResolution.y;
    vec3 color = vec3(0.5, 0.5, 0.5);
    
    // 计算当前像素到原点的距离
    float d = length(uv);
    
    // 设置比较对象，这里即圆的半径
    float r = 0.3;
    color *= smoothstep(r, r + 0.01, d);
    
    fragColor = vec4(color, 1.0);
}
```

1. **如果没有保持长宽比的话，可能显示的是一个椭圆，因为图形被拉伸了！**
2. 这里的smoothstep这一行可以理解为：小于r，返回0.0值，color乘以0.0等于0.0，显示黑色；大于r，则返回1.0.保持color值，既保持背景色

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.2mnd30zvrw80.webp)

​                                                                                         结果图

- **极坐标**

​         这一方面主要用来生成齿轮，花瓣等沿着圆周变化的形状，因为是以角度为自变量，所以atan()函数基本上一定会用到(当我们看到别人的shader里面有atan函数时，可以往角度方向去想一下)

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    // 将坐标原点移到中心
    uv = uv - 0.5;
    // 保持原来的长宽比
    uv.x *= iResolution.x / iResolution.y;
    vec3 color = vec3(0.5, 0.5, 0.5);
    
    // 计算当前像素到原点的距离
    float d = length(uv);
    float r = 0.0;
    
    // ****************   added  *************
    // 设置比较对象，这里添加角度的变化项
    r = 0.3 + 0.1 * cos(6.0 * atan(uv.y, uv.x));
    
    // 对应第二个图
    // r = 0.3 + 0.1 * abs(cos(6.0 * atan(uv.y, uv.x)));
    // 对应第三个图
    // r = 0.3 + 0.1 * smoothstep(-.5, 0.4, cos(8.0 * atan(uv.y, uv.x)));
    // ****************   added  *************
    
    
    color *= smoothstep(r, r + 0.01, d);
    
    fragColor = vec4(color, 1.0);
}
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.4r38h4j83720.webp" alt="image" style="zoom:67%;" />   <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.25hsp6rmak80.webp" alt="image" style="zoom:67%;" />     <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.6l552wxuyp00.webp" alt="image" style="zoom:67%;" />

​                 图一                                                   图二                                                      图三

##### 1.2.2 图形的基本操作

主要包括平移，旋转，缩放

旋转矩阵

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.1iy25plo5nvk.webp)

**注意：**

​        缩放，旋转是有参考点的，下面的两个矩阵都是针对原点进行计算的，如果不在原点，需要先平移到原点，然后进行旋转，缩放操作，然后再平移到原先的位置。

```glsl
#define PI 3.14159265359

// 旋转矩阵
mat2 rotate2d(float _angle){
    return mat2(cos(_angle),-sin(_angle),
                sin(_angle),cos(_angle));
}

// 缩放矩阵
mat2 scale(vec2 _scale){
    return mat2(_scale.x,0.0,
                0.0,_scale.y);
}
   
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    // 将坐标原点移到中心
    uv = uv - 0.5;
    // 保持原来的长宽比
    uv.x *= iResolution.x / iResolution.y;
    
    // ****************   added  *************
    // 平移
    uv += (sin(iTime), cos(iTime)) * 0.2;
    // 旋转
    // uv *= rotate2d( sin(iTime)*PI );
    // 缩放
    // uv *= scale( vec2(0.8 + abs(sin(iTime)) * 0.5) );
    // ****************   added  *************
    
    vec3 color = vec3(0.5, 0.5, 0.5);
    
    // 计算当前像素到原点的距离
    float d = length(uv);
    float r = 0.0;
    r = 0.3 + 0.1 * smoothstep(-.5, 0.4, cos(8.0 * atan(uv.y, uv.x)));
    color *= smoothstep(r, r + 0.01, d);
   
    fragColor = vec4(color, 1.0);
}
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/1.58bk40zkisw0.gif" alt="1" style="zoom: 33%;" />               <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/2.43s81uxpc900.gif" alt="2" style="zoom:33%;" />              <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/3.2970omykd3gg.gif" alt="3" style="zoom:33%;" />  

​            图一                                                                图二                                                           图三

##### 1.2.3 重复，分形

- **重复**

​        如果我们绘制多个相同的图案，那么可以采取下面的想法：我们的画布是归一化到0.0到1.0，如果我们将画布划分若干份，然后将每一部分的坐标重新映射到0.0到1.0，那么我们在一个画布上绘制的图案就会被同步到其它区域上，从而实现了重复图形的绘制。

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;

    // ****************   added  *************
    // 实现重复部分的代码    
    uv = uv * 3.0;
    uv = fract(uv);
    // ****************   added  *************  
      
    // 将坐标原点移到中心
    uv = uv - 0.5;
    // 保持原来的长宽比
    uv.x *= iResolution.x / iResolution.y;
    
    vec3 color = vec3(0.5, 0.5, 0.5);
    
    // 计算当前像素到原点的距离
    float d = length(uv);
    float r = 0.0;
    
    r = 0.3 + 0.1 * smoothstep(-.5, 0.4, cos(8.0 * atan(uv.y, uv.x)));
    
    color *= smoothstep(r, r + 0.01, d);
    
    fragColor = vec4(color, 1.0);
}
```

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.28tbyxrx6tus.webp)

- **分形**

主要用于模拟树木，山脉，树叶等等具有自相似的物体

```glsl
    vec2 complex_sqr(vec2 pos)
{
    return vec2(pow(pos.x, 2.0) - pow(pos.y, 2.0), pos.x * pos.y * 2.0);
}


void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord/iResolution.xy;

    vec2 c = vec2(-0.8, cos(iTime) * 0.2);
    vec2 z = vec2(uv.x * 2.0 - 1.0, uv.y - 0.5) * 2.0;
    float iterations = 0.0;
    while (length(z) < 20. && iterations < 50.)
    {
        z = complex_sqr(z) + c;
        iterations += 1.;
    }
    vec3 col = vec3(1. - iterations * 0.02);
    
    fragColor = vec4(col,1.0);
}
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/4.10eklxiyfou8.gif" alt="4" style="zoom:50%;" />                                <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.38hljol6hle0.webp" alt="image" style="zoom:95%;" />

##### 1.1.4 随机

​         通过上面的一些方法，我们可以生成一些规则的图形，如齿轮，花瓣等等；规则代表着一种美，数学之美，而无序则代表着自由，有无限的想象空间，比如天上的云朵，跳动的火苗，海平面溅起的浪花等，这些自然之物无法用数学公式来把它们限制在一个方程式之下。那只能用魔法去打败魔法了，是随机去模拟大自然的随机，noise就是这门艺术。

> [shader noise](https://nashpan.github.io/2021/09/04/shader-noise/#more)



#### 1.3 image

##### 1.3.1 贴图

这个比较简单，就是将现有的图片 导入到 我们的画布中，使用函数texture2D即可

```glsl
// 导入你的图片，下面的路径是我的文件路径
#iChannel0 'file://pictures/46.jpg'

void main()
{
    vec2 uv = gl_FragCoord.xy / iResolution.xy;
    
    // 对你的纹理进行采样，获取对应像素处的颜色
    vec4 color = texture2D(iChannel0, uv);
    
    // 输出到屏幕，即为原图
    gl_FragColor = color;
}
```

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.31u4tqtqfya0.webp)   <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/2e339c48-1d52-4342-a954-8a983f4ac731.png" alt="1" style="zoom:37%;" />

​                     原图                                                             运行代码图



##### 1.3.2 玩转图片

- **图片转场**

说到图片，大家最先想到的特效是啥？我首先想到的是各种转场

> https://gl-transitions.com/gallery

```glsl
#iChannel0 'file://pictures/1.png'
#iChannel1 'file://pictures/46.jpg'

float SQRT_2 = 1.414213562373;
float dots = 20.0;
vec2 center = vec2(0, 0);

vec4 transition(vec2 uv) {
  bool nextImage = distance(fract(uv * dots), vec2(0.5, 0.5)) < ( fract(iTime * 0.4) / distance(uv, center));
  return nextImage ? texture2D(iChannel0, uv) : texture2D(iChannel1, uv);
}

void main()
{
    vec2 uv = gl_FragCoord.xy / iResolution.xy;
    vec4 color = vec4(0.0);
    color = transition(uv);
    gl_FragColor = color;
}
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/5.79leqhys3280.gif" alt="5" style="zoom: 33%;" />

- **制作精灵动画**

​          这一部分是为了说明，对于一幅图片，我们可以只选择其中一部分进行采样；(ps: 文字渲染的一种方式是将所有文字信息离屏渲染到一张纹理上，使用时去查找对应的部分进行渲染）

```glsl
#iChannel0 'file://pictures/1.png'

int col = 5;
int row = 4;

void main() {
    vec2 uv = gl_FragCoord.xy / iResolution.xy;
    vec4 color = vec4(0.0);

    vec2 fRes = iResolution.xy / vec2(float(col), float(row));
    vec2 nRes = iResolution.xy / fRes;
    uv = uv / nRes;

    float timeX = iTime * 1.5;
    float timeY = floor(timeX / float(col));
    vec2 offset = vec2 (floor(timeX) / nRes.x, 1. - (floor(timeY) / nRes.y));

    uv = fract(uv + offset);

    color = texture2D(iChannel0, uv);

    gl_FragColor = color;
    
    
    // 进阶：第三个是抖音上比较火的网格分割图片，配合音乐的卡点
    //      来显示图片，从而产生比较有趣的效果
    
    
    //     我们可以基于此的思想 来将我们的众多图片按照不同时间
    //  绘制到画布上形成动画！！！
          
}
```

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.1mbgs1elp11c.webp" alt="image" style="zoom:80%;" />               <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/6.1q8gsml6aexs.gif" alt="6" style="zoom: 30%;" />                   <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/7.1bqb88wkd89s.gif" alt="7" style="zoom: 67%;" />

​                     原图                                                         运行代码图                                                抖音特效



##### 1.3.3 图片操作

- **反转图片**

​          下面这个图片看起来好像有点阴间的感觉，反转操作可用于通道中颜色较小的值反转成较大的值，然后用于处理，或者相反，是一种数据处理的思想（类似于将数据重新映射，类比于笛卡尔坐标到极坐标，或者从极坐标映射到笛卡尔坐标系），常用的颜色处理还有颜色空间的转换，如RGB颜色空间转换到HSV颜色空间，都是利用了这种思想

```glsl
// 导入你的图片，下面的路径是我的文件路径
#iChannel0 'file://pictures/46.jpg'

void main()
{
    vec2 uv = gl_FragCoord.xy / iResolution.xy;
    
    // 注意这里，这里我只取rgb通道的颜色，因为a通道不用反转
    vec3 color = texture2D(iChannel0, uv).rgb;
    
    // 进行反转的操作
    color = 1.0 - color;
  
    gl_FragColor = vec4(color, 1.0);
}
```

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.5tx13g88hs40.webp)                               <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.1oqer3erzu74.webp" alt="image" style="zoom:95%;" />

​               原图                                                                                     效果图



- **图片灰度化**

​        记住这个数字 035911， 即0.3， 0.59， 0.11 (大概是这个比例，有的是0.299, 0.587, 0.114 或者其它，都是按照这个基准进行灰度化的，PS软件的灰度图就是按照这个比例进行灰度化的）

```glsl
// 导入你的图片，下面的路径是我的文件路径
#iChannel0 'file://pictures/46.jpg'

void main()
{
    vec2 uv = gl_FragCoord.xy / iResolution.xy;
    
    // 注意这里，这里我只取rgb通道的颜色，因为a通道不用反转
    vec3 color = texture2D(iChannel0, uv).rgb;
    
    // 进行灰度的处理
    float grey = color.r * 0.3 + color.g * 0.59 + color.b * 0.11;
    color = vec3(grey);
  
    gl_FragColor = vec4(color, 1.0);
}
```

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.7dsnegk2zr80.webp)                              ![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.2rd7bnt6ut8.webp)

​                      原图                                                                               效果图

- 颜色的加减乘除

​          这一块对应于AE 或者PS中的 叠加方式，有Add, Substract, Multiply, Average, Difference等等，因为这一块没有发现比较好玩的效果，暂时不深入，一般常用的就是Add方式就可，即colorA + colorB的方式。

##### 1.3.4 卷积，算子

- 模糊操作

  主要用到的是**高斯模糊，**其它的还有均值模糊，径向模糊等等，各有各的用途，按照需求选择合适即可。

> [高品质后处理：十种图像模糊算法的总结与实现](https://zhuanlan.zhihu.com/p/125744132)

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.3sghfjkaxgo0.webp)

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.7ghxkhwd4q80.webp)

- 边缘检测

​          边缘的表现形式如下图：**通常是通过寻找图像的梯度来检测边界**，有两种代表算法，sobel算子（基于一阶导数的）和laplacian算子（基于二阶导数）。

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.3nrpt49koba0.webp)

1.   **sobel 算子**	

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.2j86nqc9x020.webp)

对于不连续的函数，一阶导数可以写作：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.694l71jtb4k0.webp)

 或

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.2nfbs2zop7m0.webp)

所以有：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.6wq57o3xzs.webp)

 假设要处理的图像为I，在两个方向求导：

 水平变化：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.gvqie82f40o.webp)

垂直变化：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.7ce4yv3c1800.webp)

 结合以上两个结果求出：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.2aaekdlimp1c.webp)

 统计极大值所在的位置，就是图像的边缘。

```glsl
#iChannel0 'file://pictures/pixel.png'

void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    vec3 col;
    
    /*** Sobel kernels ***/
    // Note: GLSL's mat3 is COLUMN-major ->  mat3[col][row]
    mat3 sobelX = mat3(-1.0, -2.0, -1.0,
                       0.0,  0.0, 0.0,
                       1.0,  2.0,  1.0);
    mat3 sobelY = mat3(-1.0,  0.0,  1.0,
                       -2.0,  0.0, 2.0,
                       -1.0,  0.0,  1.0);  
    
    float sumX = 0.0;   // x-axis change
    float sumY = 0.0;   // y-axis change
    
    for(int i = -1; i <= 1; i++)
    {
        for(int j = -1; j <= 1; j++)
        {
            // texture coordinates should be between 0.0 and 1.0
            float x = (fragCoord.x + float(i))/iResolution.x;   
            float y =  (fragCoord.y + float(j))/iResolution.y;
            
            // Convolve kernels with image
            sumX += length(texture( iChannel0, vec2(x, y) ).xyz) * float(sobelX[1+i][1+j]);
            sumY += length(texture( iChannel0, vec2(x, y) ).xyz) * float(sobelY[1+i][1+j]);
        }
    }
    
    float g = abs(sumX) + abs(sumY);
    //g = sqrt((sumX*sumX) + (sumY*sumY));
    
    if(g > 1.0)
        col = vec3(1.0,1.0,1.0);
    else
        col = col * 0.0;
    
    fragColor.xyz = col;
}
```

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.6ttyltgldm80.webp)

​                                                   原图                                                                                  运行算法图

2.**laplacian算子**

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.3dytl7kab920.webp)

对于不连续函数的二阶导数是：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.3g446js1c5u0.webp)

因此使用的卷积核是：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.4gytix99njs0.webp)

### 2. Shader 和 AI 

#### 2.1 特效相关

- 瘦脸特效，大眼特效，头部晃动特效等等

​        **这里会使用AI的方法提取人物的关键点**，如下图中的左右太阳穴的关键点，下巴的关键点，人体头部的关键点等等，然后再加上图像处理的方法，就会产生很多有趣的效果！！！

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.704q25ekllc0.webp" alt="image" style="zoom: 50%;" />        <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/8.373haivhp600.gif" alt="8" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/9.6oga687i1xc0.gif" alt="9" style="zoom: 50%;" />       <img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/10.2g0l6csob89w.gif" alt="10" style="zoom: 58%;" />



- 背景特效

​         这里就会用到AI中的人像分割技术，将人物抠出来，然后背景部分将其灰度化，人物部分保留原始的颜色即可。

![11](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/11.pzbqp854000.gif)

#### 2.2 DLSS

> ​	[DLSS 2.0 - 重新定义AI渲染](https://zhuanlan.zhihu.com/p/116211994)

​          DLSS简单来说就是我们使用传统的图形学方法渲染一帧低清晰度的图片，然后使用AI的方法将其“复原”成高分辨率的图片，即图片的高清修复。因为在真实感方面，光线追踪技术渲染的真实感很好，但是在高分辨率下，计算量爆炸；有了DLSS后，我们可以使用光线追踪技术渲染低分辨率的图片，然后使用AI进行高清修复。

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-hosing/image.4j53kcu7cew0.webp" alt="image" style="zoom:80%;" />



### 参考资料

- [浅谈生成艺术](https://zhuanlan.zhihu.com/p/426092206)
- [the book of shaders](https://thebookofshaders.com/)
- [shadertoy](https://www.shadertoy.com/)
- [iq大神博客](https://iquilezles.org/)
- https://www.shadertoy.com/view/Ms2SD1