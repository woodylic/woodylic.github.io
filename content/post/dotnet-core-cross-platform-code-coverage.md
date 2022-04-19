---
title: .net core跨平台代码覆盖分析
date: 2018-04-28 18:26:42
tags: [".net core", "test", "CI/CD"]
---


一个常见的CI流程，build -> unit test -> coverage analysis -> package。.net core实现了跨平台，但由于工具链缺失的原因，test和code coverage只能在Windows下完成。最近工作上需要配置，发现工具链已经补充完成，基本能工作了。现把配置流程记录下来。

<!--more-->

## 准备试验项目

准备一个简单的计算器library项目和对应的xunit项目：

```
CoreCalc
  |-- src
      |-- CoreCalc.Lib
  |--test
      |-- CoreCalc.Lib.UnitTest
```

验证build和test正常工作：

```sh
# ~/CoreCalc$
dotnet build
```

```sh
# ~/CoreCalc$
dotnet test test/CoreCalc.Lib.UnitTest/CoreCalc.Lib.UnitTest.csproj \
  --no-restore --no-build
```

如有多个测试项目，可以另建一个*.Test.sln文件，把测试项目加进去，然后对改sln执行测试。

## 输出测试结果

`dotnet test` 的默认输出是console，如果需要把测试结果publish到Jenkins等CI系统，需要先把测试结果保存下来。[Logger Extensions](https://github.com/Faizan2304/LoggerExtensions)项目提供了此功能，通过指定test的logger参数，把测试结果记录到文件中，目前支持AppVeyor, NUnit和xUnit3三种格式。

下面以xUnit Logger为例：

1. 为测试项目添加XunitLogger

```sh
# ~/CoreCalc/test/CoreCalc.Lib.UnitTest$
dotnet add package XunitXml.TestLogger --version 2.0.0
```

2. 执行测试的时候加入logger参数

```sh
# ~/CoreCalc
dotnet test test/CoreCalc.Lib.UnitTest/CoreCalc.Lib.UnitTest.csproj \
  --no-restore --no-build \
  --logger:xunit
```

测试结果保存到测试项目的TestResults文件夹下。

## 统计代码覆盖率

.Net以往开源的coverage选择貌似只有OpenCover一个，Windows only。今年出现了一个新的选择，[Coverlet](https://github.com/tonerdo/coverlet)，支持跨平台，（目标）支持多种格式的report。

使用方法也很简单：

1. 为测试项目添加coverlet

```sh
# ~/CoreCalc/test/CoreCalc.Lib.UnitTest$
dotnet add package coverlet.msbuild --version 1.1.1
```

2. 执行测试的时候加入logger参数

```sh
# ~/CoreCalc
dotnet test test/CoreCalc.Lib.UnitTest/CoreCalc.Lib.UnitTest.csproj \
  --no-restore --no-build \
  -p:CollectCoverage=true \
  -p:CoverletOutputFormat=opencover \
  -p:CoverletOutputDirectory=../../artifacts/coverage
```

## 生成代码覆盖报告

通常和[OpenCover](https://github.com/OpenCover/opencover)一起配合使用的是[ReportGenerator](https://github.com/danielpalme/ReportGenerator)，目前alpha版已经支持跨平台了，可以直接选用。

使用方法如下：

1. 为测试项目添加ReportGenerator

```sh
# ~/CoreCalc/test/CoreCalc.Lib.UnitTest$
dotnet add package ReportGenerator --version 4.0.0-alpha3
```

注：这里纯粹是为了dotnet restore的时候会下载ReportGenerator而已，代码并不会引用。

2. 生成报告：

```sh
# ~/CoreCalc
dotnet ~/.nuget/packages/reportgenerator/4.0.0-alpha3/tools/ReportGenerator.dll \
  "-reports:artifacts/coverage/coverage.xml" \
  "-targetdir:artifacts/coverage"
```

可以看到覆盖率报告生成到了artifacts/coverage下。
