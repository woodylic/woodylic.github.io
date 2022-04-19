---
title: 把db scheme脚本移出resources目录
date: 2017-03-02 00:00:50
tags: ["maven"]
---

在[用DbUnit实现Dao层的单元测试](https://github.com/woodylic/blog/issues/2)中，我把数据库scheme脚本放在了/src/test/resources目录下，这样在初始化dataSource的时候，可以很方便地初始化数据库：
```xml
<jdbc:initialize-database data-source="dataSource" ignore-failures="DROPS">
  <jdbc:script location="classpath:scheme.sql"/>
</jdbc:initialize-database>
```

但是scheme.sql放在/src/test/resources下其实不太合理，数据库的scheme脚本应该是project级别的，不应该放在test下。而放在/src/main/resources下感觉也不太合理，/src/main/resources下放置的内容应该是在运行时会用到的各种配置文件。

<!--more-->

参考[what's the recommened location for sql ddl scripts](http://stackoverflow.com/questions/7686334/whats-the-recommended-location-for-sql-ddl-scripts)，我把scheme.sql移到了/src/main/db。对应地，在单元测试初始化数据库的时候，script location就不能用classpath来定位了。把spring配置修改一下，以${project.basedir}开始，定位ddl脚本。
```xml
<jdbc:initialize-database data-source="dataSource" ignore-failures="DROPS">
  <jdbc:script location="file:///${project.basedir}/src/main/db/scheme.sql"/>
</jdbc:initialize-database>
```

再次执行测试，失败了。Cannot resolve ${project.basedir}。${project.basedir}是maven的变量，Spring是无法直接读取的。但是可以通过properties文件来传递。具体的步骤如下：

1. 为测试创建一个/src/test/resources/test-env.properties文件。
```properties
project.basedir = ${project.basedir}
```

2. 在pom里配置对test-env.properties文件使用filter。这样在maven处理test resources的时候，会把${project.basedir}替换为变量值。
```xml
<build>
  <testResource>
    <directory>src/test/resources</directory>
    <filtering>true</filtering>
  </testResource>
</build>
```

3. 在Spring配置文件中引用test-env.properties，然后使用project.basedir属性定位scheme文件。
```xml
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
  <property name="locations">
    <array>
      <value>classpath:test-env.properties</value>
    </array>
  </property>
</bean>

<jdbc:initialize-database data-source="dataSource" ignore-failures="DROPS">
  <jdbc:script location="file:///${project.basedir}/src/main/db/scheme.sql"/>
</jdbc:initialize-database>
```

重新执行测试，顺利通过。