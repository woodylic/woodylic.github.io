---
title: 使用maven和flyway管理数据库schema
date: 2017-03-23 23:58:41
tags: ["maven", "flyway"]
---

# 为什么需要数据库schema版本管理

作为开发人员，代码版本控制已经属于标配了，但是很多团队还不会把数据库schema纳入版本控制。在开发期间，schema随着需求变化，而这些变化难以追踪记录，从而导致开发人员之间，以及各个环境下的数据库schema的一致性问题。

引入数据库schema版本管理，是期望模仿代码版本控制，把schema的所有版本记录下来，并且可以自动地把数据库升级到最新版本。

*参考：[Why database migrations?](https://flywaydb.org/getstarted/why)*

<!-- more -->

# liquibase和flyway

liquibase和flyway都是数据库schema版本管理工具。liquibase提供了更强大的功能，比如多数据库支持，rollback等。而flyway没有引入过多的概念，使用更为简单。在够用的情况下，使用flyway学习成本更低。

# flyway基础

flyway要求为每一次数据库schema变更创建一个sql语句，以VN__（双下划线）作为前缀（比如V1 __create_table_A.sql, V2__create_index_on_field_1。）

配置好数据库连接，执行flyway migrate命令，flyway会自动执行需要的脚本（比如在空数据库执行，会顺序执行V1和V2两个脚本，而在和V1版本一致的数据库执行，只会执行V2脚本），把数据库更新到最新状态。

在第一次执行flyway migrate的时候，flyway会在数据库中建立一张默认命名为schema_version的表，用于跟踪该数据库实例的schema版本变更。

除了migrate，flyway还支持clean, validate, baseline等命令。具体可以查看[flyway document](https://flywaydb.org/documentation/).

# 通过maven使用flyway

flyway支持命令行，java api，maven，gradle等各种方式调用。我接触的项目使用maven进行管理，所以选择了通过maven使用flyway的方式。

## 项目结构

下面是在标准的maven文件夹结构中，增加了/src/main/db/migration，用于存放数据库脚本。

```java
src
|--main
   |--db
      |--migration
   |--java
   |--resources
|--test
pom.xml
```

## maven集成flyway

把以下节点放入pom.xml。大多数情况下，flyway配置中使用的数据库驱动依赖和数据库连接，与dependency/db.properties里的值是一样的，所以本例假设在pom.xml中统一定义，在flyway配置中直接引用。

```xml
<build>
    <plugins>
        <!-- 通过mvn flyway:migrate命令自动migrate数据库。 -->
        <plugin>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-maven-plugin</artifactId>
            <version>${flywaydb.version}</version>
            <dependencies>
                <!-- 配置flyway使用的数据库驱动 -->
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>${mysql.driver.version}</version>
                </dependency>
            </dependencies>
            <configuration>
                <!-- 配置目标数据库连接 -->
                <driver>${db.driver}</driver>
                <url>${db.url}</url>
                <user>${db.user}</user>
                <password>${db.password}</password>
                <!-- 配置数据库脚本位置，本里中对应/src/main/db/migration -->
                <locations>
                    <location>${db.migration.location}</location>
                </locations>
                <!-- 设置sql脚本文件的编码 -->
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 使用flyway同步数据库

假设使用场景如下：

- Day 1：项目启动，开发A和B把数据库schema大致定，开始工作。

A或B需要在db/migration目录建立数据库schema脚本，命名为V1__initialize.sql，checkin到svn。A和B获取最新代码，并执行migrate命令，初始化本地数据库（并创建flyway版本控制表）。

```bash
mvn flyway:migrate
```

- Day 2：A在开发中需要修改table1，B在开发中需要新建table2，对应的修改应该先反映到migration下：

```java
db/migration
   |--V1__initialize.sql        //创建于Day 1
   |--V2__update_table_1.sql    //A创建于Day 2
   |--V3__add_table_2.sql       //B创建于Day 2
```

然后A和B获取最新代码，重新执行flyway migrate，得到对方的更新。

*以上场景，A和B有可能都把文件前缀命名为V2（因为不知道对方需要更新），这里需要A和B协调后重命名其中一个为V3。*

## 使用Spring和flyway自动初始化数据库

可以设置在Spring启动的时候，自动检测数据库版本，如果不是最新的，则自动更新。但一般生产环境应该不会使用这种，这种能力，一般应该是在自动化测试里面使用，每次自动化测试前，使用flyway创建一个新的数据库。

首先需要在pom添加flyway-core依赖。

```xml
<!-- 测试启动的时候，使用flyway自动初始化数据库。 -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>${flywaydb.version}</version>
    <scope>test</scope>
</dependency>
```

然后修改spring-dao-test配置，声明在flywayBean加载的时候，执行migrate操作。

```xml
<!-- 使用flyway初始化数据库 -->
<bean id="flyway" class="org.flywaydb.core.Flyway" init-method="migrate" depends-on="dataSource">
    <property name="dataSource" ref="dataSource"/>
    <property name="encoding" value="UTF-8"/>
    <property name="locations" value="${db.migration.location}"/>
</bean>
```

# 项目实践思考

我还没在正式项目中使用过数据库schema版本管理，预计以后会按以下方法实践：

(1) 先在开发过程使用flyway，暂不考虑用于生产环境数据库管理。

在开发过程中，每次修改的可能是一些小的变动。而在部署生产环境的时候，应更倾向于把某个开发周期的所有变更合并成一个部署文件。生产环境一般需要跳板机连接，跳板机上未必允许安装flyway。另外flyway维护的话，需要创建schema_version表，生产环境也可能不允许。更重要的是，在升级生产环境数据库的时候，需要考虑不影响现有数据，这在开发周期容易被忽略。

(2) 版本号定义和生产环境部署脚本。

参照(1)，项目中的db文件夹下的内容可能如下：

```java
db
  |--migration
     |--V1.0.1__create_table_1.sql
     |--V1.0.2__create_table_2.sql
     |--V1.1.1__create_table_3.sql
     |--V1.1.2__create_table_4.sql
  |--deployment     
     |--v1.0_database_change.sql
     |--v1.1_database_change.sql
```

其中migration里面的内容是每次数据库更新的相应脚本，deployment里面的脚本，在每个版本部署前，人工（通过migration脚本整合，或者从测试数据库导出）创建，修改并测试后用于部署生产环境。

在本例中，产品的两个迭代，版本分别为v1.0和v1.1，migration脚本的版本则比产品版本多一级。

(3) migration脚本checkin后不应再被修改。

根据flyway的机制，如果数据库的schema版本已经是最新，则不会执行更新。如果脚本checkin后，被其他开发使用过了，再次修改，那么新的修改内容将无法同步给已执行过该脚本的数据库。

考虑到数据库schema修改相对频度较低，开发人员应该考虑清楚，设计相对稳定后，再checkin数据库migration脚本。

# 参考

[快速掌握和使用Flyway](http://blog.waterstrong.me/flyway-in-practice/)
[Java敏捷数据库迁移框架](http://www.cnblogs.com/humeng126/p/3662704.html)