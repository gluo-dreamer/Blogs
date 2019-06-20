# Qt-应用程序目录结构
---
在Linux下一个Qt应用程序最后的软件包目录因该大致为下图所示：
``` shell
application_name/
├── bin
├── lib
├── share
│   ├── application_name
│   │   ├── config
│   │   ├── database
│   │   └── translations
│   └── doc
└── var
```

各个目录说明：
+ bin目录下存放可执行的二进制文件
+ lib目录下存放库文件
+ config目录下存放配置文件
+ database目录下存放数据库文件
+ translations目录下存放翻译文件
+ doc目录下存放软件文档
+ var目录下存放软件运行日志 （可有可无）