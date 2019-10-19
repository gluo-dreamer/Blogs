# QImage隐式共享
QImage是Qt中最常见的图像数据容器。

一个从图像文件中加载图像数据到QImage简单例子
```C++
QImage image;
image.load(QLatin1String("test.bmp"));
```
得到的QImage可以与QPainter一起使用，也可以传递个小部件。
现在，如果我们的图像数据来自其他地方怎么办（非文件）？例如，QPainter制作的自定义图形、相机、计算机视觉框架（cv::Mat)等。
其实，我们可以在以下几个构造函数中找到答案。
```C++
QImage(const QSize &size, QImage::Format format)

QImage(int width, int height, QImage::Format format)

QImage(uchar *data, int width, int height, QImage::Format format, QImageCleanupFunction cleanupFunction = nullptr, void *cleanupInfo = nullptr)

QImage(const uchar *data, int width, int height, QImage::Format format, QImageCleanupFunction cleanupFunction = nullptr, void *cleanupInfo = nullptr)

QImage(uchar *data, int width, int height, int bytesPerLine, QImage::Format format, QImageCleanupFunction cleanupFunction = nullptr, void *cleanupInfo = nullptr)

QImage(const uchar *data, int width, int height, int bytesPerLine, QImage::Format format, QImageCleanupFunction cleanupFunction = nullptr, void *cleanupInfo = nullptr)

QImage(const char *const [] xpm)

QImage(const QString &fileName, const char *format = nullptr)

QImage(const QImage &image)

QImage(QImage &&other)
```
现在，我们来看一些常见的用例，以及这些构造函数如何满足每种用例的特定需求。

### 案例1：QImage拥有的图像数据
即时生成图像的常见情况是使用QPainter及光栅绘制引擎绘制到QImage中：
```C++
QImage image(800, 480, Format_ARGB32_Premultiplied);
image.fill(Qt::transparent);
QPainter p(&image);
p.fillRect(10, 10, 50, 50, Qt::red);
p.end();
```
在这里，底层图像数据用**image**本身分配和拥有。
>  涉及到传递给Qt Quick图片格式(Format)，基本建议如下：
>  + 通常，首选Format_ARGB32_Premultiplied。因为它是栅栏绘制引擎的首选格式之一，QPainter在定位QImage时使用该格式，Premultiplied格式非常适合Qt Quick，因为场景图形渲染器及其材料依赖于alpha来实现，在为我们的图像创建纹理时，默认的基于OpenGL的Qt Quick场景图将避免对ARGB32_Premultiplied图像进行任何可能昂贵的QImage格式转换。
>  + 当完全不需要Alpha通道时，Format_RGB32时一个很好的选择。从这样的QImage创建OpenGL场景图形纹理时，无需要进行格式转换。

其他格式有时会导致对converToFormat()函数的调用。这是这是要尽可能避免的，最好从一开始就选择正确的格式。

值得注意的是，为了获得正确的按Qt-on-CPU转换，OpenGL实现必须支持GL_EXT_bgra，GL_EXT_texture_format_BGRA8888或某些特定于供应商的变体。这可能与GLES规范未要求BGRA支持的较旧的，仅OpenGL ES 2.0的系统有关。在没有这些（A）RGB32图像数据的情况下，内部将需要一个额外的麻烦步骤（这是快速的，几乎所有旧的Qt 4代码都会这样做），或者转换为按字节排序的QImage格式（某些较新的格式更喜欢） QtGui和其他地方的代码），因为（A）RGB32格式不是按字节排序的。

但是按字节顺序排列的是Qt 5.2中引入的Format_RGB（A | X）8888 [_Premultiplied]格式。这些很好，因为当读取为字节时，在小端系统和大端系统上的顺序都是R，G，B，A，这意味着可以使用GL_RGBA格式和GL_UNSIGNED_BYTE数据将图像数据传递给glTex（Sub）Image2D调用。原样输入。

> + 当必须以较旧的OpenGL ES 2.0系统为目标时，自定义OpenGL渲染代码（例如，在QOpenGLWindow，QOpenGLWidget，QQuickFramebufferObject等内部）可以受益于使用字节格式（如Format_RGBA8888）的QImage，这仅仅是因为要减少代码量写。（缺少BGRA支持时，无需进行手动处理或额外的QImage格式转换）
> + QtGui的某些此类代码可能使用的OpenGL帮助程序（最著名的是QOpenGLTexture）在使用QImage时也更喜欢Format_RGBA8888，并将启动其他格式的转换。

因此，使用Format_ARGB32_Premultiplied或Format_RGB32的较早的默认建议不一定总是有效。然而，在Qt Quick的背景下，坚持使用这些格式通常仍然是正确的选择。

### 案例2：包装外部只读图像数据
将外部引擎获取的图像数据封装为QImage的常用解决方案时使用带有**const uchar指针的QImage构造函数**
```C++
void *data = ......
QImage image(static_cast<const uchar *>(data), 800, 480, QImage::Format_RGB32);

//image 不拥有数据
//"data"必须保持有效，直到 “image”生效为止
```
这在里，QImage在构造时不进行数据内存分配和副本的拷贝，由应用出现确定图像宽度、高度、每行字节数来反应原始图像数据。

从const uchar *创建的实例具有特殊的意义，即在任何修改QImage的尝试都将实现深拷贝（拷贝副本），而无视应用计数。
```C++
const uchar *data = ......
QImage wrapper(data, ...);
wrapper.setPixelColor(5, 5, Qt::green);
// 这里 wrapper 将不是一个封装器，它将拷贝一个副本
```
在实践中通常我们不可避免的要复制数据，例如，当与OpenCV接口使用时，将CV_8UC4图像转换为QImage代码可能如下所示：
``` C++
QImage mat8ToImage(const cv::Mat &mat) {
switch (mat.type()) {
case CV_8UC4: {
	QImage wrapper(static_const<const uchar *>(mat.data), mat.cols, mat.rows, static_cast<int>(mat.step), QImage::Format_RGB32);
	return wrapper.rgbSwapped();
	
}
}
}
```
在这里，我们使用const或非const指针构造函数都时没关系的。由于rgbSwapped()函数本身就进行数据复制复制

### 案例3：包装和修改外部非const图像数据
使用uchar指针的构造函数，正如non-const参数所暗示的那样，这允许通过包装QImage修改外部非拥有的数据，而无需复制图像数据。
```C++
void *data = ...
QImage image(static_cast<uchar *>(data), 800, 480, QImage::Format_RGB32);
QPainter p(&image);
p.fillRect(80, 80, 90, 90, Qt::red);
p.end();
//注意：只要image有效，就必须保持data一直有效
```
这在里，我们在现有内容的顶部绘制一个红色的矩形。这可能时由于QImage的一个可能令人困惑的功能：在引用计数为1（即不在多个QImage实例之间共享图像数据）的情况下，非拥有的，非只读的数据修改不会导致深拷贝。

以下演示了在开始使用QImage类时可能遇到的一个潜在的问题：
```C++
void drawStuff_Wrong(QImage image)
{
	QPainter p(&image);
	p.fillRect(10, 10, 30, 30, Qt::red);
}

void *data = ....
QImage packager(static_cast<uchar *>(data), 800, 480, QImage::Format_RGB32);
drawStuff_Wrong(packager);
```
在这里，红色矩形不会出现在**data**所指向的原始图像中。由于是按值传递给函数，因此图像的引用计数增加到2，因此打开QPainter时进行的修改将导致深拷贝发生，解决方案时通过指针或引用传递。

### 案例4： 访问图像数据
最后但并非最重要的一点，我们来谈谈访问QImage的字节。
QImage类中提供了`pixel()`、`bits()`、`constBits()`、`constScanLine()`、`scanLine()`函数对QImage数据的不同访问。
```C++
QImage image(800, 480, QImage::Format_ARGB32_Premultiplied);
const uchar *p = image.constBits();
......
// 只要image有效，p就有效
```
读写访问是通过bits()和scanLine()完成。在实际中，强烈建议在需要只读访问时使用constBits()和constScanLine()，以避免意外调用non-const bits()或scanLine()并制作昂贵且完全不必要的副本图像数据
```C++
QImage image1(800, 480, QImage::Format_RGB32);
QImage image2 = image1;
const uchar *p = image1.bits();
//const uchar *p = image1.constBits();
//我们只想要读取权限，而调用non-const bits()函数导致image1发生深拷贝，浪费时间与内存。
```
> 当多个实例共享同一数据或映像包装只读外部数据(通过const uchar*构造函数)时，非const版本函数将导致深拷贝。
