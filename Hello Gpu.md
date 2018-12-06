# hello gpu

参考 : 有许多非常专业的网站提供了更加详细，全面的解释

[Learn OpenGL 主页](https://learnopengl-cn.github.io/)

[极客学院 曲面细分教程](http://wiki.jikexueyuan.com/project/modern-opengl-tutorial/tutorial30.html)

[Render Doc 官网](https://renderdoc.org/)

[DX OS 解释](https://docs.microsoft.com/en-us/windows/desktop/direct3d11/d3d10-graphics-programming-guide-output-stream-stage)

简写 : 在初次出现时会用全称，后续直接用简写代替
``` python
DrawCall              : "dc"      #一次渲染调用
# 数个着色器或阶段的简称
Input Assembler Stage : "IA"      #输入装配 阶段
Vertex-Shader Stage   : "VS"      #顶点着色 阶段
Tessellation Stage    : "TS"      #曲面细分 阶段
Geometry-Shader Stage : "GS"      #几何着色 阶段
OutputStream Stage    : "OS"      #流输出   阶段
```

# GPU
从最简单的像素颜色插值，到如今的渲染管线，是经过了几代人的努力的结果。

在过去的20年，图形硬件发生了不可思议的转变。在 1999 年，NVIDA发布了 GeForce256，于是 NVIDA 创造了GPU这个术语。

GPU从完全 Fixed-Function 到如今的 highly Programmable 的形态，经过了很多改变。

现在，为了效率考虑，有些部分仍然是 Fixed-Function，但整体的趋势是变得 Programmable and flexible。

GPU 专注于高并发的任务处理，并且拥有极快的速度。

甚至对某些功能 (rapidly accessing texture images，快速访问图片纹理)，设计了专门的芯片。

在这里，我们会谈到，GPU如何用他的高并行结构来完成渲染程序的。

有一个定义， **a shader core**，可以看成一个小的处理器，它能处理一些相对独立的任务，比如 计算一个顶点的变化，或者计算 一个片元的颜色。

在渲染时，每一秒都将有数十亿的 **shader invocations**，这指就是 shader 程序运行的实例。（一个 shader core 运行一次 shader program）

对于所有的处理器，延时是一个必须要面对的问题。相比于存储在寄存器中数据，从缓存中获取肯定更慢，而从内存中获取延时就会更加明显了。

对于GPU来说，如何处理等待数据的读取时一个很重要的话题（比如，获取一张纹理贴图）。

## 并行数据结构
```
Much of a GPU’s chip area is dedicated to a large set of processors, 
called shader cores, often numbering in the thousands.
```
GPU 的大部分芯片区域是一大堆处理器的集合，称为 shader cores。通常有上千个。

>* GPU 大规模并行处理数据
>>* 顶点数据或者像素数据，其结构是完全一样的，相似性极高
>>* 这些调用(invocatio, 这里指一次vs或者ps的执行)都是独立的。
我们在vs或者ps中，不会去获得邻近调用的信息，也没有一个共享的可写的内存。（可读倒是有)
>>* 缺点:
这样的规则会导致一些不方便的地方。举个例子，在 Learn OpenGL 教程中，我们做高斯模糊时，并不能在ps中获得邻近的点的信息。
而是做了2次dc。

![Guass Opengllearn](pic/hello_gpu/guass_opengllearn.png)

```
Say a mesh is rasterized and two thousand pixels have fragments to be processed;
```
RTR在这里有个，2000个片元待处理的例子。

>* 只有一个 shader core
>>* 它在读取纹理是，会需要内存的访问，要花上几百上千个时钟周期，这段时间，只能做单纯的等待。
>* 优化1，在等待时，我们给每个片元一点存储空间，遇到等待时切换环境
>>* 在读取纹理时，切换环境（处理下一个片元），继续执行。即，遇到需要 stall 的地方， switch。

在这种架构中，我们通过切换来隐藏延迟。通过分离数据中的指令逻辑，GPU提供了一种更进一步的设计。
SIMD （single instruction, multiple data）。通过这种形式，对固定数量的着色器程序执行相同的指令。

![Simd](pic/hello_gpu/simd.png)

在现代GPU术语中，我们的2000个片段调用的顶点着色程序的运行实例，称之为线程。

和CPU不一样的是，它包含了一小块输入内容的内存，和一部分执行指令的寄存器。
（正因为它如此相对独立，我们在切换时的代价也很低)

这些执行相同 shader 程序的线程会被捆绑为组，在NVIDA称为 wrap，在AMD中称为 wavefront。

一组线程会被一定数量的 shader core 执行，通常是 8-64 个。

在SIMD中，每个线程对于一条通道。

![3.1](pic/hello_gpu/3.1.png)

>* 2000 线程，63 组的例子
>>* 以组为单位执行指令，遇到内存读取时（图中的txr指令），交换组
>>* 缺点，如果组不够多，在第三组被执行完成后，我们还是会出现真正的延迟。
>>* 最主要的原因，是线程使用的寄存器的数量。因为寄存器的总数是有限的，
线程关联的寄存器越多，则驻留在GPU内的线程就越少，导致可以分出的组也越少。
>>* 还有一个原因就是，读取内存的频率。
>>* 以及，if或循环语句引起的动态分支。
```
However, if some threads, or even one thread, take
the alternate path, then the warp must execute both branches, throwing away the
results not needed by each particular thread [530, 945].
```
因为我们是 SIMD 模式，所以只要有一个线程有不同的分支，所有的内容都要跑一遍不同的分支。

这被称为线程分歧(thread divergence)

## 管线总览

在之前，我们可能提及到了，管线中有些功能是 Fixed-Function，一些是 Programmable,，在这里给出总览。

![3.2](pic/hello_gpu/3.2.png)

其中:
>* 绿色: 完全可编程 （programmable）
>* 黄色：可配置，不可编程（configurable）
>* 蓝色：固定功能 （Fixed-Function）
>* 虚线：可选 (Optional)
```
The logical model can help you reason about what affects performance, but it should
not be mistaken for the way the GPU actually implements the pipeline.
```
RTR提醒: 逻辑模型只是让你更好的理解，真正的GPU实现不一定如此。

## 可编程的

