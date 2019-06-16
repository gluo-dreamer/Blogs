# Qt-国际化
---

安装翻译文件的常规方法：先获取用户选择的语言，如果为空则将系统本地locale名填充，一般我们都将翻译文件以**appName_locale.qm**形式存放在translations目录下。在加载软件翻译文件的同时我们还将加载Qt库自带的对于翻译文件。
> appName: 为软件名称
> locale：用QLocale返回当前系统的域名（例如：zh_CN、en_US等）

``` C++
bool installTranslator(QApplication &app)
{
    const QString locale = userSettingLocale();
    const QString systemLocale = QLocale::system().name();
    if (locale.isEmpty()) {
        locale = systemLocale;
    }

    QTranslator appTranslator;
    QTranslator qtTranslator;
    const QString appTranslatorDir = QCoreApplication::applicationDirPath() + QLatin1QString("../share/translations");
    const QString appTranslatorFile = QString::fromLatin1("%1_%2.qm").arg(QCoreApplication::applicationName()).arg(locale);
    if (appTranslator.load(appTranslatorFile, appTranslatorDir)) {
        const QString qtTranslatorDir = QLibraryInfo::location(QLibraryInfo::TranslationsPath);
        const QString qtTranslatorFile = QString::fromLatin1("qt_%1.qm").arg(locale);
        if (qtTranslator.load(qtTranslatorFile, qtTranslatorDir)
                || qtTranslator.load(qtTranslatorFile, appTranslatorDir)) {
            app.installTranslator(&appTranslator);
            app.installTranslator(&qtTranslator);

            return true;
        }
    }

    return false;
}

QString userSettingLocale() const
{
    //TODO: 查询配置文件得到用户设置的语言。
}
```

应用程序感知可翻译为哪些语言方法：搜索存放翻译文件的目录（这里指定为**/appDir/../share/translations**下）,提取翻译文件中 locale 部分的内容并转换为对应的语言字符串。然后存放在QComboBox中。
``` C++
void GeneralSettings::fillLanguageBox() const
{
    const QString currentLocale = userSettingLocale();

    //系统语言
    m_page->languageBox->addItem(QLatin1String("Chinese"), QLatin1String("C"));
    if (currentLocale.isEmpty() || currentLocale == QLatin1String("C")) {
        m_page->languageBox->setCurrentIndex(m_page->languageBox->count() - 1);
    }

    const QString creatorTrPath =
            QCoreApplication::applicationDirPath() + QLatin1String("../share/translations");
    const QStringList languageFiles =
            QDir(creatorTrPath).entryList(QStringList(QString::fromLatin1("%1*.qm").arg(QCoreApplication::applicationName())));

    Q_FOREACH(const QString &languageFile, languageFiles)
    {
        int start = languageFile.indexOf(QLatin1Char('_'))+1;
        int end = languageFile.lastIndexOf(QLatin1Char('.'));
        const QString locale = languageFile.mid(start, end-start);
        // no need to show a language that creator will not load anyway
        if (hasQmFilesForLocale(locale, creatorTrPath)) {
            QLocale tmpLocale(locale);
            QString languageItem = QLocale::languageToString(tmpLocale.language()) + QLatin1String(" (")
                                   + QLocale::countryToString(tmpLocale.country()) + QLatin1Char(')');
            m_page->languageBox->addItem(languageItem, locale);
            if (locale == currentLocale)
                m_page->languageBox->setCurrentIndex(m_page->languageBox->count() - 1);
        }
    }
}

static bool hasQmFilesForLocale(const QString &locale, const QString &appTrPath)
{
    static const QString qtTrPath = QLibraryInfo::location(QLibraryInfo::TranslationsPath);

    const QString trFile = QLatin1String("/qt_") + locale + QLatin1String(".qm");
    return QFile::exists(qtTrPath + trFile) || QFile::exists(appTrPath + trFile);
}
```