---
title: 在maven中使用profile和filtering管理多环境配置
date: 2017-02-23 23:46:46
tags: ["maven"]
---

# Filtering

Filtering是maven的resource插件提供的功能，作用是用环境变量、pom文件里定义的属性和指定配置文件里的属性替换属性文件(*.properties)里的占位符(如${jdbc.url})

# Profile

<profile>是pom文件里的一个xml元素，在profile里几乎可以定义所有在pom里的定义的内容(如depencency，build，properties等，但不能再定义profile)。当一个profile被激活时，它定义的<dependencies>，<properties>等就会覆盖掉原pom里定义的相同内容，从而可以通过激活不同的profile来使用不同的配置。

<!--more-->

# Filtering + Profile管理不同环境的配置

为每个环境定义profile，以及对应的属性(properties)。构建时通过激活相应的profile，用其中的属性去替换application.properties里的占位符。

## 定义profile
```xml
<project>
  <profiles>
    <profile>
      <id>dev</id>
      <properties>  
        <!-- 定义active.profile变量 -->
        <active.profile>dev</active.profile>
      </properties>
      <activation>
        <!-- 作为默认profile -->
        <activeByDefault>true</activeByDefault>
      </activation>
    </profile>
    <profile>
      <id>prod</id>
      <properties>
        <!-- 定义active.profile变量 -->
        <active.profile>prod</active.profile>
      </properties>
    </profile>
  </profiles>
</project
```

## 定义properties文件
 
文件结构如下：
```
/main
  /filters
    /db.dev.properties   #dev环境配置
    /db.prod.properties  #prod环境配置
  /resources
    /db.properties       #真正被程序使用的配置文件
```

/resources/db.properties文件的内容如下：
```
dataSource=${db.dataSource}
driverClassName=${db.driverClassName}
url=${db.url}
username=${db.username}
password=${db.password}
```

/filters/db.dev.properties文件内容如下：
```
db.dataSource=org.apache.commons.dbcp2.BasicDataSource
db.driverClassName=com.mysql.jdbc.Driver
db.url=jdbc:mysql://127.0.0.1:3306/todolist-db?useUnicode=true&characterEncoding=utf8
db.username=root
db.password=123qwe
```

/filters/db.prod.properties内容类似，不再贴出。

## 在pom定义build行为
```xml
<build>
  <filters>
    <!-- 指定filter位置，注意路径里用到了第一步定义的profile属性 -->
    <filter>src/main/filters/db.${active.profile}.properties</filter>
  </filters>
  <resources>
    <!-- 定义resources位置，注意filtering要设为true -->
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>
    </resource>
  </resources>
</build>
```

## 为不同环境构建
```
mvn clean package -Pprod  #为prod环境构建
mvn clean package         #没有指定profile，默认为dev环境构建
```