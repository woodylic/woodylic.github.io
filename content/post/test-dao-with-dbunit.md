---
title: 用DbUnit实现Dao层的单元测试
date: 2017-02-22 14:37:53
tags: ["test", "dbunit"]
---

# 前言

Dao层的单元测试，测试目的在于：

- 验证数据连接环境（包括ORM的配置和Dao的实现）的正确性。
- 验证sql语句的正确性。

单元测试的常规步骤是：
1. 准备测试数据。
2. 执行待测方法。
3. 检查执行结果。

<!-- more -->

#1通常需要初始化db数据，#3通常也是检查数据库结果。

在每个开发人员使用独立数据库的前提下，[DbUnit](http://dbunit.sourceforge.net/)提供了一套控制数据库状态的方法，简化#1和#3的工作。而[Spring Test DbUnit](https://springtestdbunit.github.io/spring-test-dbunit/)则提供了Spring和DbUnit集成，允许开发人员使用annotation而不是继承来完成数据库的setup和teardown。

# 待测项目介绍

整个测试基于[todo-list](https://github.com/woodylic/todo-list)，个人为巩固Spring MVC + MyBatis框架搭建的练习项目。

该项目的数据库只有一张数据表(tbl_todos)，结构如下：

Field | Type | Comments
---|---|---
id | bigint (20) | 自增主键
title | varchar(50) |
description | varchar(500) |
status | tinyint(1) | 0：未完成 1：已完成

Dao接口包含几个最常见的CRUD方法，结构如下：
```java
public interface TodoDao {
    int insert(TodoItem todoItem);
    int update(TodoItem todoItem);
    int deleteByPrimaryKey(Long id);
    TodoItem selectByPrimaryKey(Long id);
    List<TodoItem> selectAll();
}
```
# 测试用例设计

默认CRUD的测试用例比较简单，也不是本文重点，略。

# 测试数据准备

DbUnit支持使用xml文件来定义测试数据（包括初始数据和期望结果），在本例中，预期在初始化数据库的时候，插入两条数据，测试数据文件定义如下：

```xml
<!-- basic-test-data.xml -->
<dataset>
  <tbl_todos id="1" title="Professional C#" description="4th version" status="0" />
  <tbl_todos id="2" title="Core Java" description="Volume I" status="0" />
</dataset>
```
*id其实是作为自增id存在的，但是DbUnit确实可以把id控制为需要的值，可以把id改为3，4做试验。不确定是否只在h2 database里生效。*

insert测试，预期插入一条数据，期望结果如下：
```xml
<!-- expected-result-for-insert.xml -->
<dataset>
  <tbl_todos id="1" title="Professional C#" description="4th version" status="0" />
  <tbl_todos id="2" title="Core Java" description="Volume I" status="0" />
  <tbl_todos id="3" title="Item Title" description="Item Description" status="0" />
</dataset>
```

update和delete的期望结果类似，不在此列出。

# 单元测试环境配置

Dao层的实现依赖Spring，单元测试也会以相同的方式获得待测Dao对象。需要准备一个专门用于Dao层单元测试的Spring配置文件。
```xml
<!-- spring-dao-test.xml -->
<beans>

  <!-- 导入spring-dao -->
  <import resource="classpath:spring-dao.xml" />

  <!-- 覆盖原dataSource，使用h2数据库做单元测试 -->
  <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="org.h2.Driver" />
    <property name="url" value="jdbc:h2:mem:todo-list;DB_CLOSE_DELAY=-1;MODE=MySQL" />
  </bean>
  <jdbc:initialize-database data-source="dataSource" ignore-failures="DROPS">
    <!-- 执行下面script完成初始化 -->
    <jdbc:script location="classpath:db/scheme.sql"/>
  </jdbc:initialize-database>
</beans>
```

一般来说用于测试的spring配置文件应该尽可能保证和项目的spring配置一致，所以通常会import项目spring配置，然后把需要修改的bean覆盖掉。本例中把数据库从MySQL改为h2，以便其他团队成员不需要做前置配置即可执行unit test。但实际上借助DbUnit，用MySQL作为单元测试的依赖也很方便。

# 编写单元测试

```java
@RunWith(SpringJUnit4ClassRunner.class)                         //指定使用Spring Test
@ContextConfiguration(value="classpath:spring-dao-test.xml")    //加载Spring Context
@TestExecutionListeners({
        DependencyInjectionTestExecutionListener.class,         //为了让Spring处理DbUnit的annotation，
        DbUnitTestExecutionListener.class })                    //必须使用的ExecutionListener
public class TodoDaoTest {

    @Autowired
    private TodoDao todoDao;
    
    @Test
    @DatabaseSetup(value = {"basic-test-data.xml"})             //指定数据库初始化数据
    public void testSelectAll() throws InterruptedException {
    	List<TodoItem> todoItems = todoDao.selectAll();
        assertEquals(2, todoItems.size());
    }
    
    @Test
    @DatabaseSetup(value="basic-test-data.xml")                 //指定数据库初始化数据
    @ExpectedDatabase(value="expected-result-for-insert.xml")   //指定数据库期望结果
    public void testInsert() throws InterruptedException {
    
        TodoItem todo = new TodoItem();
        todo.setTitle("Item Title");
        todo.setDescription("Item Description");
        todo.setStatus(0);
    
    	int impactRows = todoDao.insert(todo);
    	assertEquals(1, impactRows);
    }
    
    //其他测试方法类似，略。
}
```

# 注意
1. 由于Dao依赖Spring生成，需要配置RunWith以及ContextConfiguration。如果不用，那么需要自己写代码读取配置文件，加载Spring Context。
2. 使用Spring Test DbUnit，必须配置好DbUnitTestExecutionListener及其依赖DependencyInjectionTestExecutionListener。
3. @DatabaseSetup在执行的时候会清空数据库，再插入初始化数据。如果期望数据库在测试方法结束回滚，需要添加下面代码：
```java 
@DatabaseTeardown(value="basic-test-data.xml", DatabaseOperation.DELETE) //删除data-set里的数据。
```
4. 编写testInsert的时候，原来打算在空数据库内插入数据，预期结果是数据表里面有一条id=1的数据。但是发现JUnit在执行的时候，会优先执行SELECT测试，把basic-test-data.xml的数据加载。Teardown会把数据删除，但不是恢复数据库状态，所以在执行INSERT的时候，生成的id是3而不是1。考虑到Unit Test Case之间不允许相互依赖，在这个例子中，哪怕其他方法都做了Teardown，insert也应该做DatabaseSetup，获得一个稳定的前置状态。
