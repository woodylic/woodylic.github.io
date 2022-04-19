---
title: slf4net和NLog集成
date: 2017-05-31 00:10:27
tags: ["log", "NLog", "slf4net"]
---

这是一个slf4net和NLog集成的简单例子。https://github.com/woodylic/logging-facade-samples/

关于为什么用slf4net，可以参照这篇文章：
[第三方组件引用另一个第三方组件的悲剧](http://www.cnblogs.com/blqw/p/3726493.html)。简单地说，就是解耦组件对特定log library的依赖，交由组件的调用方决定。

<!--more-->

# 项目结构

```
Slf4Net.Sample
    |-- Slf4Net.Sample.Library
    |-- Slf4Net.Sample.App
```

其中Slf4Net.Sample.Library是组件，Slf4Net.Sample.App是程序入口，调用Slf4Net.Sample.Library实现功能。

# 实现Slf4Net.Sample.Library

该项目只包含一个Class, SomeComponent。该类包含一个方法DoSomething，在方法起止位置分别写log。

```CSharp
public class SomeComponent
{
    private static slf4net.ILogger _logger = slf4net.LoggerFactory.GetLogger(typeof(SomeComponent));

    public void DoSomething()
    {
        _logger.Info("Before executing ...");
        
        Console.WriteLine("Executing DoSomething");
        
        _logger.Info("After executing ...");
    }
}
```

该项目只依赖slf4net，logger类型为slf4net.ILogger，通过slf4net.LoggerFactory获得。具体使用哪个log library，对此项目没有任何影响。

# 实现Slf4Net.Sample.App

App项目作为程序入口，只是对Library项目的简单调用。

```CSharp
class Program
{
    static void Main(string[] args)
    {
        new Slf4Net.Sample.Library.SomeComponent().DoSomething();
    }
}
```

# 在Slf4Net.Sample.App配置日志模块

对日志功能的实现，依赖slf4net，slf4net.NLog和NLog。通常通过配置方式声明：

```xml
<configuration>
  <configSections>    
    <section name="slf4net" type="slf4net.Configuration.SlfConfigurationSection, slf4net" />
    <section name="nlog" type="NLog.Config.ConfigSectionHandler, NLog" />
  </configSections>
  <!-- slf4net选择NLog作为Log实现 -->
  <slf4net>
    <factory type="slf4net.NLog.NLogLoggerFactory, slf4net.NLog" />
  </slf4net>
  <!-- NLog的配置 -->
  <nlog>
    <targets>
      <target type="Console" name="simpleLog" encoding="UTF-8" layout="${date:format=HH\:mm\:ss.fff} [${level}] ${message}" />
    </targets>
    <rules>
      <logger name="*" minlevel="Info" writeTo="simpleLog" />
    </rules>
  </nlog>
</configuration>
```

通常而言日志的配置比较长，推荐放到外部文件NLog.config。这样App.config的nlog节点需要修改为

```xml
<nlog configSource="NLog.config" />
```

通过nuget安装NLog.Config和NLog.Schema，会自动创建NLog.config和NLog.xsd，前者提供一个简单的NLog模板，后者实现了智能提示，辅助开发人员更方便地编写NLog配置。

# 调试

如果日志功能无效，可以通过在nlog节点打开internalLog（默认是Off，可以改为Debug），通过查看异常信息排查。

```xml
<nlog internalLogLevel="Debug" internalLogFile="c:\temp\nlog-internal.log">
```

# 参考资料

- [slf4net Configuration](https://github.com/ef-labs/slf4net/wiki/Configuration)
- [Nlog wiki](https://github.com/NLog/NLog/wiki)

# 更新

Common.Logging是另一个logging facade，比slf4net活跃，使用方法和slf4net类似。Sample已经加入源码。另外.NET Core也加入了类似的接口。感觉使用微软定义的接口，然后按需求挑选开源实现是个不错的方向。