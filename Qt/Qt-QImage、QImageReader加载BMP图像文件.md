# QImage、QImageReader加载BMP图像文件
最近一个项目遇到使用QImage无法加载一张270多M位图。经查阅源码发现限制如下: `quint64(bi.biWidth) * qAbs(bi.biHeight) > 16384 * 16384`, 以下为查找过程：

```C++
bool QImage::load(const QString &fileName, const char* format)
{
    *this = QImageReader(fileName, format).read();
    return !isNull();
}
```
在这里我们知道QImage使用load加载本地文件时使用QImageReader类实现。

```C++
bool QImageReader::read(QImage *image)
{
    ......

    const bool result = d->handler->read(image);

    ......

}
```
我们要知道Qt是通过插件形式实现各种图片格式文件支持，有一个很重要的抽象类`QImageIOHandler`, 通过继承并实现read、write虚函数实现文件读取写入，查到QImageReader::read()函数，`const bool result = d->handler->read(image);`其中`handler`成员变量就是一个QImageIOHandler指针。因为我读取的图像为bmp位图，通过查找到QBmpHandler(此类继承QImageIOHandler类)。
