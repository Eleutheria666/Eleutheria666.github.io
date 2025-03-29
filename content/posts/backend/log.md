---
title: slfj+logback源码解析
date: 2024-10-04
tags: ["SSM","Java","code analyse"]
aliases: ["/analyse/logback"]
---



SLF4j是一个日志门面，类似于JDBC，提供Java日志记录的统一接口

Logback是一个日志实现，类似于mysql-connector，实现SLF4j提供的接口以实现日志打印功能。

类似的日志实现还有Log4j、JDK Logging等。但由于这些日志实现实现的不是SLF4j提供的接口，因此需要使用转换器转换至SLF4j的接口

![图片](http://mmbiz.qpic.cn/mmbiz_png/KyXfCrME6ULAiaQPr6M7tZ08qNJMSX8F9yktVnzoUhibpHtXmFx9bCqCyCjXOo1AA6djBRE8oWGx2XWicOfZlibeYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Logback 核心模块主要包括以下部分：

- **Core**：处理日志事件的核心逻辑
- **Classic**：基于 SLF4J 提供的接口实现日志记录功能
- **Access**：用于 Web 应用程序中的 HTTP 访问日志记录



# Logger的获取

Java项目中获取Logger示例：

```JAVA
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger logger = LoggerFactory.getLogger(Example.class); // Class获取
private static final Logger logger = LoggerFactory.getLogger("example"); // String获取
logger.info("This is an info message");
```

### LoggerFactory的创建

首先，通过`LoggerFactory.getLogger()`获得`Logger`对象

1. 返回底层日志实现的`ILoggerFactory`对象，若未初始化，先初始化日志实现的`ILoggerFactory`
2. 获取特定`ILoggerFactory`生产的Logger对象

```java
public static Logger getLogger(String name) {
    ILoggerFactory iLoggerFactory = getILoggerFactory();  // 初始化底层日志实现的ILoggerFactory
    return iLoggerFactory.getLogger(name); // 获取特定ILoggerFactory生产的Logger对象
}

public static Logger getLogger(Class<?> clazz) {
    Logger logger = getLogger(clazz.getName());  // 本质还是调用String方式获取Logger
    if (DETECT_LOGGER_NAME_MISMATCH) {
      // 类命和调用类的名称一致性检查
    }
    return logger;
}
```

ILoggerFactory接口定义了日志门面实现的Logger工厂要干的事：只要生产Logger即可

```java
public interface ILoggerFactory {
    public Logger getLogger(String name);
}
```

Logger接口规范Logger如何记录日志

```java
public void debug(String format, Object arg1, Object arg2);
public void debug(String format, Object... arguments);
public void debug(String msg, Throwable t);
```

------

获取日志实现的`ILoggerFactory`对象

**slf4j委托具体实现框架的StaticLoggerBinder来返回一个ILoggerFactory，从而对接到具体实现框架上**

```java
public static ILoggerFactory getILoggerFactory() {
    // 先初始化
    if (INITIALIZATION_STATE == UNINITIALIZED) {
        synchronized (LoggerFactory.class) {
            if (INITIALIZATION_STATE == UNINITIALIZED) {
                INITIALIZATION_STATE = ONGOING_INITIALIZATION;
                performInitialization();
            }
        }
    }
    
    // 初始化后或已经初始化则直接获取初始化的ILoggerFactory对象
    switch (INITIALIZATION_STATE) {
        case SUCCESSFUL_INITIALIZATION:
            return StaticLoggerBinder.getSingleton().getLoggerFactory();  // 关键
        case NOP_FALLBACK_INITIALIZATION:
            return NOP_FALLBACK_FACTORY;
        case FAILED_INITIALIZATION:
            throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
        case ONGOING_INITIALIZATION:
        }
    throw new IllegalStateException("Unreachable code");
}
```

### Logger的创建







# Logger打印日志

这里以调用`Logger.debug(String)`为例，调用时会最终调用`buildLoggingEventAndAppend()`，将日志记录请求构造成日志事件 `LoggingEvent`，传递给对应的 appender 处理

```JAVA
// ch.qos.logback.classic的Logger
public void debug(String msg) {
    filterAndLog_0_Or3Plus(FQCN, null, Level.DEBUG, msg, null, null);
}

private void filterAndLog_0_Or3Plus(final String localFQCN, final Marker marker, final Level level, final String msg, final Object[] params, final Throwable t) {
    ...  // Filter过滤器
    buildLoggingEventAndAppend(localFQCN, marker, level, msg, params, t);
}

private void buildLoggingEventAndAppend(final String localFQCN, final Marker marker, final Level level, final String msg, final Object[] params, final Throwable t) {
    LoggingEvent le = new LoggingEvent(localFQCN, this, level, msg, t, params); // 构造日志事件 
    le.setMarker(marker);
    callAppenders(le); // 将LoggingEvent传送给当前logger及其父类的所有appender
}
```

Appender 持有 Filter 和 Encoder 的引用，可以分别对日志进行过滤和格式转换。

```java
public interface Appender<E> extends LifeCycle, ContextAware, FilterAttachable<E> {
    String getName();
    void setName(String name);
    void doAppend(E event) throws LogbackException;
}
```

Appender接口有两个实现类

- `UnsynchronizedAppenderBase`，不同步
- `AppenderBase`，同步，doAppend方法加了synchronized

![logback_Appender_UML](https://img2018.cnblogs.com/blog/1731892/202001/1731892-20200131175854442-1184777290.png)

获取当前Logger的appenderlist，并调用每个appender的`doAppend`方法实现日志记录

```java
public void callAppenders(ILoggingEvent event) {
    int writes = 0;
    for (Logger l = this; l != null; l = l.parent) {
        writes += l.appendLoopOnAppenders(event);
        // 当前Logger的additive属性为false，表明不再访问其父Logger
        if (!l.additive) {
            break;
        }
    }
    if (writes == 0) {
        loggerContext.noAppenderDefinedWarning(this);
    }
}

public int appendLoopOnAppenders(E e) {
    int size = 0;
    final Appender<E>[] appenderArray = appenderList.asTypedArray();
    final int len = appenderArray.length;
    for (int i = 0; i < len; i++) {
        appenderArray[i].doAppend(e);
        size++;
    }
    return size;
}
```

`UnsynchronizedAppenderBase`的`doAppend`方法只是对Appender进行状态和过滤检查，最终还是调用具体实现类的`append`方法记录



参考文献

[一个著名的日志系统是怎么设计出来的？ (qq.com)](https://mp.weixin.qq.com/s/XiCky-Z8-n4vqItJVHjDIg)

[slf4j和logback源码解读 - 敬敬不想造轮子 - 博客园 (cnblogs.com)](https://www.cnblogs.com/ljno/p/15205554.html#:~:text=slf4j源)

[SpringBoot | 第二十五章：日志管理之自定义Appender - oKong_趔趄的猿 - 博客园 (cnblogs.com)](https://www.cnblogs.com/okong/p/springboot-twenty-five.html#logback自定义Appender)