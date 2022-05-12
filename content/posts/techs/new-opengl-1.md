---
title: "OpenGL 学习笔记 1"
date: 2021-10-27T11:30:03+00:00
description: "三角形的初体验"
# weight: 1
# aliases: ["/first"]
tags: ["OpenGL"]
categories: "图形学与图形API"
#showToc: true
#TocOpen: false
draft: false
#hidemeta: false
#comments: false
#canonicalURL: "https://canonical.url/to/page"
#disableHLJS: false
#hideSummary: false
#searchHidden: false
#ShowPostNavLinks: true
#cover:
#    image: "<image path/url>" # image path/url
#    alt: "<alt text>" # alt text
#    caption: "<text>" # display caption under cover
#    relative: false # when using page bundles set this to true
#    hidden: true # only hide on current single page
#editPost:
#    URL: "https://github.com/Cardidi/my-page/content"
#    Text: "在 Github 上查看原Markdown"
#    appendFilePath: true
---

我志向学习游戏客户端开发，在此之前已经有了2年左右的业余Unity开发经验。现在升上大学，我想我也应该有时间去学习更为进阶的内容了。因此我准备通过自学OpenGL来写一个小的游戏引擎，来学习Shader与游戏引擎构架。

由于学习笔记对于我来说是对我不容易记住的东西进行补充说明与个人理解，因此不会说的非常详细，仅仅是对参考资料的一个补充。

话不多说，下面就是我的笔记正文。

# 参考资料

目前参考资料是：

 - LearnOpenGL CN 一个非常好用的OpenGL入门教程网站，必看。
 - OpenGL Wiki OpenGL的官方文档，够正的。

# OpenGL是什么

OpenGL是一个图形库（而且也仅仅是一个图形库，没有输入、声音和窗口管理的库函数），其本质是通过一套开源标准，让硬件与软件得以连接的API。现代显卡都支持OpenGL API。

OpenGL API本质上是一个状态机，因此我们需要在OpenGL每一次绘图之前对OpenGL的参数进行配置，才能让图形如我们所愿。（因此OpenGL也提供了一种参数的简记手段）

由于OpenGL本质上是一套开源标准，因此显卡真正的运行时API与OpenGL API必定不同，OpenGL需要在运行时对实际调用显卡的API进行重定向。这就导致了OpenGL的运行效率会有损失。

# GLFW与GLAD

之前提到过OpenGL是没有Input和Audio之类的功能，甚至连在操作系统下创建窗口的功能都没有，因此我们需要额外的库来做支持。GLFW就是做这些事情的一个库了。GLFW实现了系统环境支持，为应用提供了一套很好用的库，关键是它也是跨平台的。

那么GLAD是干什么的呢？GLAD的功能在于为OpenGL提供了重定向功能，使之可以调用到显卡中具体的操作函数（上面提到过了！）

# 开发环境配置

我之前没有学过C++，因此我这次尝试使用C++来学习OpenGL。学习OpenGL的同时也学会了C++，岂不美哉。

我具体的开发环境是macOS Big Sur(arm64) + Clang + Clion。我尝试使用教程中的做法来自己编译运行库，但是最后终归失败。

最后我使用了homebrew来导入运行库：
```bash
brew install glfw
```
安装完成后，我们可以在 /opt/homebrew/Cellar/ 下找到我们的glfw。

接下来按照LearnOpenGL的说法，去 https://glad.dav1d.de 下载glad后，构建项目目录结构如下所示：

```
ProjectRoot
 |- CMakeLists.txt
 |- src // 放置本项目源代码
 |- includes // 放置头文件
 |- libs // 放置引用库
```

将glad解压缩后，分别把内容中的include和lib放在我们项目中的对应目录。接下来，我们编写CMakeList如下所示：

```cmake
cmake_minimum_required(VERSION 3.20)
project(LearnOpenGL)

set(CMAKE_CXX_STANDARD 11)

# 设置变量
set(GLFW_HOME "/opt/homebrew/Cellar/glfw/3.3.4")

# 设置头文件目录
include_directories("${GLFW_HOME}/include")
include_directories(/include)

# 添加 GLFW3 预编译库
add_library(glfw SHARED IMPORTED)
SET_TARGET_PROPERTIES(glfw PROPERTIES IMPORTED_LOCATION "${GLFW_HOME}/lib/libglfw.3.3.dylib")

add_executable(LearnOpenGL src/main.cpp "libs/glad.c")

target_link_libraries(LearnOpenGL glfw "-framework OpenGL")
```

在Clion中更新CMakeLists文件之后，没有报错就没有问题了。接下来我们可以开始编写三角形的代码了。

（其实我觉得，glad也可以放在系统里面，作为系统的库来引用会更加方便一些）

# 制作三角形的的一些备注

下面就是一些我容易搞错的部分了，也是我学习的时候半天反应不过来的东西。

## 函数怎么区分

在使用OpenGL的过程中，我们会用到三种前缀的函数：

```c
glXXX();     // 真正的OpenGL函数
glfwXXX();   // 这个是glfw的函数
gladXXX();   // 这个是glad的函数
```

要记忆这三种函数前缀和其对应功能我觉得还是蛮简单的：gl只管绘图工作、其他的什么窗口创建和管理和输入处理之类的都是glfw的工作、而glad是管理OpenGL重定向工作，因此只会使用到一次 ```gladLoadGLLoader()``` 。

## glGenXXX 到底生成了个啥东西

在文档里面，```glGenXXX()```是用来创建OpenGL对象的一个通用方法，比如使用```glGenBuffers()```来创建缓冲对象，```glGenVertexArrays()```来创建顶点数组对象。但问题是，这俩东西生成在哪儿？

这个时候不得不提到CPU和GPU的数据传输问题了。这个是个老生常谈的问题，其基本描述是：如果图形数据一直都要保持在CPU的内存中，那么每次GPU绘制图形的时候，都要先让CPU来发送数据到GPU，GPU才能得到数据，这导致了GPU读取数据的最快速度取决于CPU发送数据能有多快，因此产生了瓶颈。

为了解决这个瓶颈，我们提出了缓冲区。通过将需要绘制的顶点数据（大量的）提前提交到GPU的内存中，使得GPU得以减少与CPU通讯量，从而提高计算速度。

这个问题清楚了，```glGenBuffers()```是什么就很好解释了。```glGenBuffers()```实质上就是在显存里面开辟一块内存空间后，将这个空间的地址（可能也不是地址）交给CPU，让CPU知道缓冲区的位置在哪里，方便CPU写数据。

换言之，这个调用类似于在GPU的内存上申请了一块空间，并且给GPU返回了这块内存的地址（类似于返回一个显存上的指针）。

```glGenVertexArrays()```理论上应该会和```glGenBuffers()```一致，只是它的类型不是Buffer而已。我总感觉有没有可能会有不一样的地方，但是我还没查到（大概率没有吧）。

## Vertex Array Object、Buffer Object 与 Attribute Pointer 的关系

Vertex Array Object是一种用来记录Buffer和Attribute Pointer的数据体。有点听不懂，我也有点迷，所以要仔细补充一下我的理解，这个是重点。

Buffer是缓冲区，Attribute Pointer是用来解释缓冲区数据是怎么放的说明（也就是属性）。我们的GPU和CPU一样是傻瓜，必须得有人给他细说一下我们给了它什么东西，它才知道怎样才能正确的读取我们给他的数据。我们在给GPU喂Buffer的时候，就必须要给他喂“食用指南”Attribute Pointer。

这里先要提一嘴：Buffer实际上还可以存储颜色数据和贴图数据等东西，为了方便GPU取数据，我们可以让顶点数据后面紧贴颜色和贴图的数据。因此我们必须告知GPU什么地方是顶点数据，什么地方是颜色数据等等。

我们知道像顶点、颜色和贴图之类的数据，都要通过渲染管线才能变成我们显示器上的像素颜色值。渲染管线里面的Vertex Shader是渲染的开始，因此我们可以知道一件事情，Buffer里面的数据先要走过Vertex Shader才能继续被处理。我们可以猜测到Vertex Shader不仅仅要处理顶点数据，它可能还要给其他的数据进行处理。因此，我们需要找到一个办法把两边的数据连接起来。

这个时候我们就要祭出Layout了。

Layout出现在了Vertex Shader和定义Attribute Pointer（也就是函数glVertexAttribPointer）两个地方。在Vertex Shader中，Layout指的是这个数据是从哪里输入的；在Attribute Pointer里面，Layout指的是数据要输给谁。一个头一个尾，拼在一起刚刚好。

因此总结，我们给Buffer定义Attribute Pointer来告知OpenGL如何读取和解释Buffer里面的数据给Shader。

这个时候我们回到Vertex Array Object，这玩意就是用来存放我们这一对Buffer和Attribute Pointer的地方。因为VAO名字里面有Array，可推断VAO可以存储很多对Buffer和Attribute Pointer。

这个时候配上LearnOpenGL的图，感觉突然好懂了一些：

![](/images/vertex_array_objects_ebo.png)

## VAO的结构示意图

等下，还有一个问题：再上图中，为什么VAO只保存了Attribute Pointer指针和EBO，没有保存Buffer的指针？因为Buffer的指针被放在了Attribute Pointer里面，我用C来描述一下就是

```c
struct VAO
{
    struct AP[16] attributePointers;
    struct EBO* elementBufferObject;
}

struct AP
{
    struct Buffer* buffer;
    // and code to guide OpenGL read buffer data
    // ...
}
```

## 不吓人的 glBindXXX

之前我们讨论过OpenGL本质上是一个状态机，因此我们需要在每一次绘图之前配置好我们“画笔”的参数，让它画出我们希望的图形。但是每一次画同样的东西都要让我重新配置OpenGL的话，是不是性能损耗太大了？

上面提到的Vertex Array Object就是用来解决这个问题的。我们预先定义好的“画笔”参数，到时候要用什么画笔直接拿来就能画，岂不美哉？

我们在绘制前，都需要调用```glBindVertexArray()```来绑定VAO与OpenGL，然后我们的OpenGL就会按VAO的信息来绘制图形。但其实我觉得一个比较讨厌的点在于，```glBindXXX()```调用的很频繁，有时候可能会弄不清楚我这个```glBindXXX()```是在干什么。所以我在这里简单补充一下LearnOpenGL的内容：

```c
// 首先，我们要先绑定VAO，才能保证后续的操作是在给我们预期的VAO操作
glBindVertexArray(VAO);

// 然后，我们绑定我们要操作的Buffer
// 需要注意的是，我们可以同时绑定Element Buffer和Array Buffer。
// 我们后续的操作对操作目标是很明确的，不用担心数据错位
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBindBuffer(GL_ELEMENT_BUFFER, EBO);

// 接下来，给Buffer塞数据
glBufferData(GL_ARRAY_BUFFER, ...);
glBufferData(GL_ELEMENT_BUFFER, ...);

// 数据塞好了，给Vertex Array Buffer定义Attribute Pointer
// 我曾疑惑为什么不在这里设定Buffer类型，但实际上这个函数只会使用到当前绑定的VAO，而不会使用到EBO
glVertexAttribPointer(...); 

// 启用刚刚做好的Attribute Pointer
glEnableVertexAttribArray(0);

// 解除当前的所有绑定，方便进行下一步操作
glBindVertexArray(0);
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindBuffer(GL_ELEMENT_BUFFER, 0);
```

# 总结

实际上，在怎样使用OpenGL上的难点主要是搞清楚VAO、VBO和EBO三者的关系。这三者搞清楚了，OpenGL API剩下的内容都不会太难（但我没说图形学简单啊），只是说会很多很杂了。