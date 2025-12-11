---
title: shader noise
date: 2021-09-04 13:00:34
tags: "shader"
categories: "算法"
---



## Noise噪声

<!-- more -->



### 什么是噪声

**噪声 = 随机 + 连续**

在图形学中，我们使用噪声就是为了把一些随机变量来引入到程序中。从程序角度来说，噪声很好理解，我们希望给定一个输入，程序可以给出一个输出：

```c++
value_type noise(value_type p) {   //简单理解为程序语言中的random即可
    ...
}
```

它的输入和输出类型的维数可以是不同的组合，例如输入二维输出一维，输入二维输出二维等。我们今天就是想讨论一下上面函数中的实现部分是长什么样的。

### 噪声的分类

根据wiki，由程序产生噪声的方法大致可以分为两类：

![noise_category](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/noise_category.png)

Perlin噪声、Simplex噪声和Value噪声在性能上大致满足：Perlin噪声(O(2^n)) < Value噪声< Simplex噪声(O(n^2))，Simplex噪声性能最好,效果也最好。



### Perlin Noise

#### 实现

概括来说，Perlin噪声的实现需要三个步骤：

1. 定义一个晶格结构，每个晶格的顶点有一个“伪随机”的梯度向量（其实就是个向量啦）。对于二维的Perlin噪声来说，晶格结构就是一个平面网格，三维的就是一个立方体网格。

2. 输入一个点（二维的话就是二维坐标，三维就是三维坐标，n维的就是n个坐标），我们找到和它相邻的那些晶格顶点（二维下有4个，三维下有8个，n维下有2的n次方个），计算该点到各个晶格顶点的距离向量，再分别与顶点上的梯度向量做点乘，得到2的n次方个点乘结果。

3. 使用缓和曲线（ease curves）来计算它们的权重和。在原始的Perlin噪声实现中，缓和曲线是s(t)=3t^2−2t^3，在2002年的论文6中，Perlin改进为s(t)=6t^5−15t^4+10t^3，这里简单解释一下，为什么不直接使用s(t)=t，即线性插值。直接使用的线性插值的话，它的一阶导在晶格顶点处（即t = 0或t = 1）不为0，会造成明显的不连续性。s(t)=3t^2−2t^3，在一阶导满足连续性，s(t)=6t^5−15t^4+10t^3，在二阶导上仍然满足连续性。
   我们可以用下面的图来表示上面的第一步和第二步：

![perlin_noise](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/perlin_noise.png)

#### Perlin噪声的实现

主要代码如下：

```C++
vec2 hash22(vec2 p)
{
    p = vec2( dot(p,vec2(127.1,311.7)),
              dot(p,vec2(269.5,183.3)));

    return -1.0 + 2.0 * fract(sin(p)*43758.5453123);
}
float perlin_noise(vec2 p)
{
    vec2 pi = floor(p);
    vec2 pf = p - pi;

    vec2 w = pf * pf * (3.0 - 2.0 * pf);

    return mix(mix(dot(hash22(pi + vec2(0.0, 0.0)), pf - vec2(0.0, 0.0)), 
                   dot(hash22(pi + vec2(1.0, 0.0)), pf - vec2(1.0, 0.0)), w.x), 
               mix(dot(hash22(pi + vec2(0.0, 1.0)), pf - vec2(0.0, 1.0)), 
                   dot(hash22(pi + vec2(1.0, 1.0)), pf - vec2(1.0, 1.0)), w.x),
               w.y);
}
```

这里的实现实际是简化版的，我们在算梯度的时候直接取随机值，而没有归一化到单位圆内。

### Value Noise

它把原来的梯度替换成了一个简单的伪随机值，我们也**不需要进行点乘操作**，而直接把晶格顶点处的随机值按权重相加即可。如下图，即只需要将蓝色矩形中四个顶点的随机值按权重相加。

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/value_noise1.png" alt="value_noise1" style="zoom:67%;" /> 

#### 实现

和Perlin噪声一样，它也是一种基于晶格的噪声，也需要三个步骤：

1. 定义一个晶格结构，每个晶格的顶点有一个“伪随机”的值（Value）。对于二维的Value噪声来说，晶格结构就是一个平面网格，三维的就是一个立方体网格。

2. 输入一个点（二维的话就是二维坐标，三维就是三维坐标，n维的就是n个坐标），我们找到和它相邻的那些晶格顶点（二维下有4个，三维下有8个，n维下有2 个），得到这些顶点的伪随机值。

3. 使用缓和曲线（ease curves）来计算它们的权重和。同样，缓和曲线可以是s(t) = 3t^2 - 2t^2,也可以是s(t)=6t^5−15t^4+10t^3（如果二阶导不连续对效果影响较大时）。

Value噪声比Perlin噪声的实现更加简单，并且需要的乘法和加法操作也更少，它只需要得到晶格顶点的随机值再把它们按权重相加即可。

#### 主要代码

```C++
float value_noise(vec2 p)
{
    vec2 pi = floor(p);
    vec2 pf = p - pi;

    vec2 w = pf * pf * (3.0 - 2.0 * pf);

    return mix(mix(hash21(pi + vec2(0.0, 0.0)), hash21(pi + vec2(1.0, 0.0)), w.x),
               mix(hash21(pi + vec2(0.0, 1.0)), hash21(pi + vec2(1.0, 1.0)), w.x),
               w.y);
}
```

### Simplex Noise

Simplex噪声的计算复杂度为O(n^2^)，优于Perlin噪声的O(2n)。而且在效果上，Simplex噪声也克服了经典的Perlin噪声在某些视觉问题。

#### 实现

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/simplex_noise.png" alt="simplex_noise" style="zoom:65%;" />



和perlin实现的区别是将上面的矩形换成了一个三角形，所以现在存在一个问题：如何找到上面图形的三个顶点，解决方案如下图：

![simplex_noise1](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/simplex_noise1.png)

将等边三角形倾斜即可转化为矩形，找到顶点后，再做逆向操作即可。

#### 主要代码

```C++
float simplex_noise(vec2 p)
{
    const float K1 = 0.366025404; // (sqrt(3)-1)/2;
    const float K2 = 0.211324865; // (3-sqrt(3))/6;

    vec2 i = floor(p + (p.x + p.y) * K1);

    vec2 a = p - (i - (i.x + i.y) * K2);
    vec2 o = (a.x < a.y) ? vec2(0.0, 1.0) : vec2(1.0, 0.0);
    vec2 b = a - o + K2;
    vec2 c = a - 1.0 + 2.0 * K2;

    vec3 h = max(0.5 - vec3(dot(a, a), dot(b, b), dot(c, c)), 0.0);
    vec3 n = h * h * h * h * vec3(dot(a, hash22(i)), dot(b, hash22(i + o)), dot(c, hash22(i + 1.0)));

    return dot(vec3(70.0, 70.0, 70.0), n);
}
```

### Wavelet Noise

当使用一个3D噪声去绘制2D纹理表面时，容易出现走样和细节丢失的情形

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/waveletNoise.png" alt="waveletNoise" style="zoom: 67%;" /> 

Wavelet Noise正是为了解决这个问题

#### 实现

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/wavelet_noise_generation.png" alt="wavelet_noise_generation" style="zoom:50%;" />         

主要经过了4个步骤：

1. 产生一个噪声图R
2. 降采样
3. 上采样
4. 将原图减去上采样的纹理图得到结果图

### Worley Noise

Worley噪声属于一种细胞噪声，就是噪声值是由随机的特征点向外扩散，最终看起来像是有一个个晶胞一样的效果。也叫网格噪声，**是基于距离场**

这种细胞噪声可以应用于模拟皮革，水面等等。（Worley噪声是Voronoi噪声的改进版）

#### 实现

把空间分割成网格（cells），每个网格对应一个特征点。另外，为避免网格交界区域的偏差，我们需要计算像素点到相邻网格中的特征点的距离。这就是 [Steven Worley 的论文](http://www.rhythmiccanvas.com/research/papers/worley.pdf)中的主要思想。最后，每个像素点只需要计算到九个特征点的距离：他所在的网格的特征点和相邻的八个网格的特征点。

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/worley_noise_principle.png" alt="worley_noise_principle" style="zoom: 67%;" /> 

#### 示例代码

```glsl
vec2 r(vec2 n)
{
 	return vec2(r(n.x*23.62-300.0+n.y*34.35),r(n.x*45.13+256.0+n.y*38.89)); 
}

float Worley2D(vec2 n,float s)
{
    n /= s;
    float dis = 1.0;
    for(int x = -1;x<=1;x++)
    {
        for(int y = -1;y<=1;y++)
        {
            vec2 p = floor(n)+vec2(x,y);
            p = r(p)+vec2(x,y)-fract(n);
            dis = min(dis, dot(p, p));        
        }
    }
    return 1.0 - sqrt(dis);
	
}

float WorleyFBM(vec2 uv)
{
	float amplitude = 0.5;
    float gain = 0.5;
    float lacunarity = 2.0;
    
    float value = 0.0;
    
    const int STEPS = 4;
    for(int i = 0; i < STEPS; i++)
    {
     	value += Worley2D(uv, 2.0) * amplitude;
        amplitude *= gain;
        uv *= lacunarity;
    }
    
    return value;
}
```



### FBM

单独一个Perlin噪声虽然也有一定用处，但是效果往往很无趣。因此，Perlin指出可以使用不同的函数组合来得到更有意思的结果，这些函数组合通常就是指通过分形叠加（fractal sum）。

- **fbm 分形噪声**

noise的叠加公式如下：
$$
noise(p) + 1/2 noise(2P) + 1/4 noise(4p) + ...
$$

```C++
//fbm 叠加分形噪声
float noise_sum(vec2 p)
{
    float f = 0.0;
    p = p * 4.0;
    f += 1.0000 * noise(p); p = 2.0 * p;
    f += 0.5000 * noise(p); p = 2.0 * p;
    f += 0.2500 * noise(p); p = 2.0 * p;
    f += 0.1250 * noise(p); p = 2.0 * p;
    f += 0.0625 * noise(p); p = 2.0 * p;

   return f;
}
```

上面叠加了5层，并把初始化采样距离设置为4，这都是可以自定义的。这种噪声可以用来模拟石头、山脉这类物体。

- **对噪声返回值进行了取绝对值操作。它使用的公式如下**：

$$
|noise(p)| + 1/2|noise(2p)| + 1/4|noise(4p)|+....
$$

它对应的代码如下：

```C++
float noise_sum_abs(vec2 p)
{
    float f = 0.0;
    p = p * 7.0;
    f += 1.0000 * abs(noise(p)); p = 2.0 * p;
    f += 0.5000 * abs(noise(p)); p = 2.0 * p;
    f += 0.2500 * abs(noise(p)); p = 2.0 * p;
    f += 0.1250 * abs(noise(p)); p = 2.0 * p;
    f += 0.0625 * abs(noise(p)); p = 2.0 * p;

    return f;
}
```

由于进行了绝对值操作，因此会在0值变化处出现不连续性，形成一些尖锐的效果。通过合适的颜色叠加，**我们可以用这种噪声来模拟火焰、云朵这些物体。**Perlin把这个公式称为turbulence（湍流？），因为它看起来挺像的。

- **在之前turbulnece公式的基础上使用了一个关于表面x分量的正选函数：**

$$
sin(x+|noise(p) + 1/2|noise(2p)+ 1/4|noise(4p)|+...)
$$

这个公式可以让表面沿着x方向形成一个条纹状的结构。Perlin使用这个公式模拟了一些大理石材质。我们的代码如下：

```C++
float noise_sum_abs_sin(vec2 p)
{
    float f = 0.0;
    p = p * 7.0;
    f += 1.0000 * abs(noise(p)); p = 2.0 * p;
    f += 0.5000 * abs(noise(p)); p = 2.0 * p;
    f += 0.2500 * abs(noise(p)); p = 2.0 * p;
    f += 0.1250 * abs(noise(p)); p = 2.0 * p;
    f += 0.0625 * abs(noise(p)); p = 2.0 * p;
    f = sin(f + p.x/32.0);

    return f;
}
```

#### 多层叠加，每层noise添加位移和旋转

> [中级Shader教程07 熔岩Lava](https://blog.csdn.net/tjw02241035621611/article/details/80048713)

通过FBM构造基本形态，在FBM添加点变化：1.每一层的移动速度不一样  2.每层的旋转不一样

相关代码

```glsl
float fbm ( in vec2 _st) {
    float v = 0.0;
    float a = 0.5;
    vec2 shift = vec2(100.0);
    // Rotate to reduce axial bias
    mat2 rot = mat2(cos(0.5), sin(0.5),
                    -sin(0.5), cos(0.50));
    for (int i = 0; i < NUM_OCTAVES; ++i) {
        v += a * noise(_st);
        _st = rot * _st * 2.0 + shift;    //***********************************注意这里乘以了一个矩阵，并且加上了一个偏移量
        a *= 0.5;
    }
    return v;
}
```

#### 域翘曲

> [[domain warping - 2002](https://www.iquilezles.org/www/articles/warp/warp.htm)]

*f(p) = fbm( p + fbm( p + fbm( p ) ) )*

![warp_fbm](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/warp_fbm.png) 

### 可平埔的噪声

目前公认比较好的一种做法，就是在2n维上生成n维可平铺的噪声。

> 这种方法是思想是，由于我们想要每个维度都是无缝的，也就是当该维度的值从0变成1的过程中，0和1之间比较是平滑过渡的，这让我们想起了“圆”，绕圆一周就是对该维度的采样过程，这样就可以保证无缝了。因此，对于二维噪声中的x轴，我们会在四维空间下的xz平面上的一个圆上进行采样，而二维噪声的y轴，则会在四维空间下的yw平面上的一个圆上进行采样。这个转化过程很简单，我们只需要使用三角函数sin和cos即可把二维采样坐标转化到单位圆上。同样，三维空间的也是类似的，我们会在六维空间下计算。这种方法不仅适用于Perlin噪声，像Worley噪声这种也同样是适合的。
>
> [【图形学】谈谈噪声](https://blog.csdn.net/candycat1992/article/details/50346469)



### Curl Noise

> [图形学中常见噪声生成算法综述](https://www.jianshu.com/p/9cfb678fbd95)

### noise 的应用

> [**How to Use Perlin Noise in Your Games**](http://devmag.org.za/2009/04/25/perlin-noise/)

> **Perlin noise can be used to blend between two textures**, as shown in following. You should use Perlin noise with very high contrast to prevent textures from looking fuzzy. The following code snippet shows how to blend two images using Perlin noise.

![noise_application](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/noise_application.png)

####  云

> [使用噪音模拟云的效果](https://zhuanlan.zhihu.com/p/70144964)
>
> [2D动态云彩](https://medium.com/@maksimtianblue/unity-shader-2d%E5%8A%A8%E6%80%81%E4%BA%91%E5%BD%A9-5f26d445a80a)

生成云的shadertoy代码如下

```glsl
const float cloudscale = 1.1;
const float speed = 0.03;
const float clouddark = 0.5;
const float cloudlight = 0.3;
const float cloudcover = 0.2;
const float cloudalpha = 8.0;
const float skytint = 0.5;
const vec3 skycolour1 = vec3(0.2, 0.4, 0.6);
const vec3 skycolour2 = vec3(0.4, 0.7, 1.0);

const mat2 m = mat2( 1.6,  1.2, -1.2,  1.6 );

vec2 hash( vec2 p ) {
	p = vec2(dot(p,vec2(127.1,311.7)), dot(p,vec2(269.5,183.3)));
	return -1.0 + 2.0*fract(sin(p)*43758.5453123);
}

float noise( in vec2 p ) {
    const float K1 = 0.366025404; // (sqrt(3)-1)/2;
    const float K2 = 0.211324865; // (3-sqrt(3))/6;
	vec2 i = floor(p + (p.x+p.y)*K1);	
    vec2 a = p - i + (i.x+i.y)*K2;
    vec2 o = (a.x>a.y) ? vec2(1.0,0.0) : vec2(0.0,1.0); //vec2 of = 0.5 + 0.5*vec2(sign(a.x-a.y), sign(a.y-a.x));
    vec2 b = a - o + K2;
	vec2 c = a - 1.0 + 2.0*K2;
    vec3 h = max(0.5-vec3(dot(a,a), dot(b,b), dot(c,c) ), 0.0 );
	vec3 n = h*h*h*h*vec3( dot(a,hash(i+0.0)), dot(b,hash(i+o)), dot(c,hash(i+1.0)));
    return dot(n, vec3(70.0));	
}

float fbm(vec2 n) {
	float total = 0.0, amplitude = 0.1;
	for (int i = 0; i < 7; i++) {
		total += noise(n) * amplitude;
		n = m * n;
		amplitude *= 0.4;
	}
	return total;
}

// -----------------------------------------------
void mainImage( out vec4 fragColor, in vec2 fragCoord ) {
    vec2 p = fragCoord.xy / iResolution.xy;
	vec2 uv = p*vec2(iResolution.x/iResolution.y,1.0);    
    float time = iTime * speed;
    float q = fbm(uv * cloudscale * 0.5);

    //ridged noise shape
	float r = 0.0;
    
    //noise shape
	float f = 0.0;
    // uv = p*vec2(iResolution.x/iResolution.y,1.0);
	uv *= cloudscale;
    uv -= q - time;    
    float weight = 0.7;
    for (int i=0; i<8; i++){
		f += weight*noise( uv );    
        uv = m*uv + time;
		weight *= 0.6;
    }
    
    f *= r + f;
  
    vec3 skycolour = mix(skycolour2, skycolour1, p.y);
    vec3 cloudcolour = vec3(1.1, 1.1, 0.9) * clouddark;
    f = cloudcover + cloudalpha*f; 
    vec3 result = mix(skycolour, clamp(skytint * skycolour + cloudcolour, 0.0, 1.0), clamp(f, 0.0, 1.0));
	fragColor = vec4( result, 1.0 );
}
```

运行上述代码的效果如下图：

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/Cloud.gif" alt="Cloud" style="zoom:50%;" />

#### 山脉

使用噪声作为高度图

![noise_heightMap](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/noise_heightMap.png)

#### 水面

使用噪声纹理作为高度图，不断修改水面的法线方向，使用和时间相关的变量对噪声纹理采样，得到水流动的效果；

#### 其它

noise的其它应用如生成细胞形态，皮革纹理，烟雾，大理石，布料，道路等，基本思想都是基于噪声贴图，使用噪声图作为基本纹理进行处理，如color mapping，height mapping, normal mapping等



### noise 的相关库

- [libnoise](http://libnoise.sourceforge.net/index.html)



## 参考资料

1. [WebGL进阶——走进图形噪声](https://juejin.cn/post/6844903862298476557)

2.https://www.shadertoy.com/view/ldscWj

3.[【图形学】谈谈噪声](https://blog.csdn.net/candycat1992/article/details/50346469)

4. [使用笔、和代码”生成“火焰](https://zhuanlan.zhihu.com/p/50418658)

5.[[一篇文章搞懂柏林噪声算法]](https://www.cnblogs.com/leoin2012/p/7218033.html)

6.[几种常见的程序化噪声纹理](https://zhuanlan.zhihu.com/p/341673601)

7.[Unity Shader - Noise 噪点图 - 实现简单山脉](https://blog.csdn.net/linjf520/article/details/100005528)

8.[Real-Time Procedural Effects](https://www.nvidia.fr/docs/IO/8343/RealTime-Procedural-Effects.pdf)

9.[图形学中常见噪声生成算法综述](https://www.jianshu.com/p/9cfb678fbd95)

10.[如何使用噪声生成复杂的纹理？](https://time.geekbang.org/column/article/267016?utm_source=related_read&utm_medium=article&utm_term=related_read)

