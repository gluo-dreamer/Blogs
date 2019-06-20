# Qt-插件开发之应用程序扩展
---
通过插件使应用程序可扩展的步骤：
- 定义一组用于与插件通讯的接口（通常为带纯虚函数的类，此类通常继承QObject）
- 使用`Q_DECLARE_INTERFACE()`宏告诉Qt元对象系统有关插件接口信息
- 在应用程序中使用`QPluginLoader`类加载插件
- 使用`qobject_cast()`判断插件是否实现给的接口

编写插件步骤：
- 声明插件类并继承插件接口类
- 使用`Q_INTERFACES()`宏告诉Qt元对象系统实现了哪些接口
- 使用`Q_PLUGIN_METADATA()`宏导出插件
- 在pro文件中构建插件

> + Q_DECLARE_INTERFACE(ClassName, Identifier) 宏注意事项：
>     - `Q_DECLARE_INTERFACE`与定义在**名称空间中的接口类**一起使用，**必须确保**`Q_DECLARE_INTERFACE`宏在名词空间外部。
>     - `Q_DECLARE_INTERFACE`宏应被用在**头文件中的接口类定义之后**。
>     - `Identifier`标识符一定是唯一的，且当接口类有改变（添加接口、删除接口、改变接口......）时，对应的`Identifier`标识符值也应当改变，通常我们在标识符中带入版本号。
> + 导出插件宏（`Q_PLUGIN_METADATA(PLUGINCLASS)`和`Q_EXPORT_PLUGIN2(PLUGIN, PLUGINCLASS)`)
>     - 在Qt5之前使用`Q_EXPORT_PLUGIN2(PLUGIN, PLUGINCLASS)`宏导出插件，Qt5中使用`Q_PLUGIN_METADATA(PLUGINCLASS)`替代。


下边代码将展示一组插件通讯接口：
``` C++
\file iplugin.h

namespace ExtensionSystem
{
class IPlugin : public QObject
{  
    Q_OBJECT

public:
    enum ShutdownFlag {  
        SynchronousShutdown = 0,
        AsynchronousShutdown
    };

public:
    explicit IPlugin(QObject *parent);
    virtual ~IPlugin();
    
public:
    virtual void initialize(const QStringList &args, QString &errorString) = 0;
    virtual void extensionInitialize() = 0;
    virtual bool delayedInitialized() = 0;
    virtual ShutdownFlag aboutToShutdown() = 0;

signals:
    void asynchronizationShutdownFinish();
};
}

#define IPlugin_iid "IPlugin/1.0.0"
Q_DECLARE_INTERFACE(ExtensionSystem::IPlugin, IPlugin_iid)


#if QT_VERSION_MAJOR >= 5
#   if defined(Q_EXPORT_PLUGIN)
#       undef Q_EXPORT_PLUGIN2
#       undef Q_EXPORT_PLUGIN
#       define Q_EXPORT_PLUGIN(Plugin)
#       define Q_EXPORT_PLUGIN2(Plugin, PluginClass)
#   endif
#else
#   define Q_PLUGIN_METADATA(PluginClass)
#endif
```

//TODO: 插件实现！