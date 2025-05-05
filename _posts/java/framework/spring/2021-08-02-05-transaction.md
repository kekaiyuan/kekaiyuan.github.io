---
typora-root-url: D:\git\kekaiyuan.github.io
layout: post
title: Java Spring 05—— AOP 的高级应用
categories: Spring
description: Java Spring 05—— AOP 的高级应用
keywords: Java, Spring
---

Java Spring 05—— AOP 的高级应用

# Spring JdbcTemplate

在 Spring 中为了更加方便的操作 JDBC，在 JDBC 的基础之上定义了一个抽象层。<br>
此设计的目的是为不同类型的 JDBC 操作提供模板方法，每个模板方法都能控制整个过程，并允许覆盖过程中的特定任务。<br>
通过这种方式，可以尽可能保留灵活性，将数据库存取的工作量讲到最低。

## 环境配置
添加 pom 依赖

pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mashibing</groupId>
    <artifactId>spring_demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.2.3.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.21</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/cglib/cglib -->
        <dependency>
            <groupId>cglib</groupId>
            <artifactId>cglib</artifactId>
            <version>3.3.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.aspectj/aspectjweaver -->
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.9.5</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/aopalliance/aopalliance -->
        <dependency>
            <groupId>aopalliance</groupId>
            <artifactId>aopalliance</artifactId>
            <version>1.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-aspects -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>5.2.3.RELEASE</version>
        </dependency>
    </dependencies>

</project>
```


----------


编写数据库连接的配置文件

dbconfig.properties
```properties
jdbc.username=root123
password=123456
url=jdbc:mysql://localhost:3306/demo
driverClassName=com.mysql.jdbc.Driver
```


----------


通过 xml 文件导入 dataSource 对象

applicationContext.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/aop
       https://www.springframework.org/schema/aop/spring-aop.xsd
">
    <context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="driverClassName" value="${jdbc.driverClassName}"></property>
    </bean>
</beans>
```


----------


测试
```java
ApplicationContext context = new ClassPathXmlApplicationContext("jdbcTemplate.xml");
DruidDataSource dataSource = context.getBean("dataSource", DruidDataSource.class);
System.out.println(dataSource.getConnection());
}
```
结果
```java
com.mysql.jdbc.JDBC4Connection@611889f4
```
成功地使用 druid 与 mysql 建立了连接。

## 添加 JdbcTemplate 对象
添加 pom 依赖

pom.xml
```xml
<!-- https://mvnrepository.com/artifact/org.springframework/spring-orm -->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-orm</artifactId>
	<version>5.2.3.RELEASE</version>
</dependency>
```


----------


在 xml 文件中添加 JdbcTemplate 对象

applicationContext.xml
```xml
...
<context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
	<property name="username" value="${jdbc.username}"></property>
	<property name="password" value="${jdbc.password}"></property>
	<property name="url" value="${jdbc.url}"></property>
	<property name="driverClassName" value="${jdbc.driverClassName}"></property>
</bean>
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
	<constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
</bean>
...
```

在创建 JdbcTemplate 对象必须传入一个 dataSource 对象。


----------


测试
```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
System.out.println(jdbcTemplate);
```
结果
```java
org.springframework.jdbc.core.JdbcTemplate@37858383
```

成功创建了 JdbcTemplate 对象

## 使用 JdbcTemplate 对象读写数据
使用 JdbcTemplate 对象读写数据非常地方便。<br>
我们只需要定义 sql 语句即可，其他细节 Spring 会帮我们自动完成。


### 插入单条数据


```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
String sql = "insert into emp(empno,ename) values(?,?)";
int result = jdbcTemplate.update(sql, 1111, "zhangsan");
System.out.println(result);
```

是不是很方便，只需要写 sql 语句，然后直接调用 JdbcTemplate 的方法执行 sql 语句即可。

### 插入批量数据


```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
String sql = "insert into emp(empno,ename) values(?,?)";
List<Object[]> list = new ArrayList<Object[]>();
list.add(new Object[]{1,"zhangsan1"});
list.add(new Object[]{2,"zhangsan2"});
list.add(new Object[]{3,"zhangsan3"});
int[] result = jdbcTemplate.batchUpdate(sql, list);
for (int i : result) {
	System.out.println(i);
}
```

### 查询单个对象


```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
String sql = "select * from emp where empno = ?";
Emp emp = jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(Emp.class), 7369);
System.out.println(emp);
```

注：Emp 是与数据表 emp 对应的实体类。

### 查询多个对象

```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
String sql = "select * from emp where sal > ?";
List<Emp> query = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Emp.class), 1500);
for (Emp emp : query) {
	System.out.println(emp);
}
```

### 执行组合函数

```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
String sql = "select max(sal) from emp";
Double aDouble = jdbcTemplate.queryForObject(sql, Double.class);
System.out.println(aDouble);
```

##### 8、使用具备具名函数的JdbcTemplate

jdbcTemplate.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/aop
       https://www.springframework.org/schema/aop/spring-aop.xsd
">
    <context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="driverClassName" value="${jdbc.driverClassName}"></property>
    </bean>
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
    </bean>
    <bean id="namedParameterJdbcTemplate" class="org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate">
        <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
    </bean>
</beans>
```

MyTest.java

```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
JdbcTemplate jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
import java.sql.SQLException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MyTest {
    public static void main(String[] args) throws SQLException {
        ApplicationContext context = new ClassPathXmlApplicationContext("jdbcTemplate.xml");
        NamedParameterJdbcTemplate jdbcTemplate = context.getBean("namedParameterJdbcTemplate", NamedParameterJdbcTemplate.class);
        String sql = "insert into emp(empno,ename) values(:empno,:ename)";
        Map<String,Object> map = new HashMap<>();
        map.put("empno",2222);
        map.put("ename","sili");
        int update = jdbcTemplate.update(sql, map);
        System.out.println(update);
    }
}
```

##### 9、整合EmpDao

jdbcTemplate.xml

```xml
    <context:component-scan base-package="com.mashibing"></context:component-scan>
```

EmpDao.java

```java
package com.mashibing.dao;

import com.mashibing.bean.Emp;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

public class EmpDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public void save(Emp emp){
        String sql = "insert into emp(empno,ename) values(?,?)";
        int update = jdbcTemplate.update(sql, emp.getEmpno(), emp.getEname());
        System.out.println(update);
    }
}
```

MyTest.java

```java
import com.mashibing.bean.Emp;
import com.mashibing.dao.EmpDao;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;

import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;

public class MyTest {
    public static void main(String[] args) throws SQLException {
        ApplicationContext context = new ClassPathXmlApplicationContext("jdbcTemplate.xml");
        EmpDao empDao = context.getBean("empDao", EmpDao.class);
        empDao.save(new Emp(3333,"wangwu"));
    }
}
```

# 声明式事务
现有某 service 类 ，调用某 dao ，执行了一系列 SQL 操作，此时如何为其添加事务支持？
```java
@Service
public class AService {

    @Autowired
    ADao aDao;

    public void aMethod(...){
		aDao.select(...);
		aDao.update(...);
		aDao.insert(...);
		aDao.delete(...);
		...
    }
}
```


总结：在事务控制方面，主要有两个分类：

编程式事务：在代码中直接加入处理事务的逻辑，可能需要在代码中显式调用beginTransaction()、commit()、rollback()等事务管理相关的方法

声明式事务：在方法的外部添加注解或者直接在配置文件中定义，将事务管理代码从业务方法中分离出来，以声明的方式来实现事务管理。spring的AOP恰好可以完成此功能：事务管理代码的固定模式作为一种横切关注点，通过AOP方法模块化，进而实现声明式事务。

##### 2、声明式事务的简单配置

Spring从不同的事务管理API中抽象出了一整套事务管理机制，让事务管理代码从特定的事务技术中独立出来。开发人员通过配置的方式进行事务管理，而不必了解其底层是如何实现的。

Spring的核心事务管理抽象是PlatformTransactionManager。它为事务管理封装了一组独立于技术的方法。无论使用Spring的哪种事务管理策略(编程式或声明式)，事务管理器都是必须的。

事务管理器可以以普通的bean的形式声明在Spring IOC容器中。下图是spring提供的事务管理器
![事务管理器](..\image\事务管理器.png)

1、在配置文件中添加事务管理器

jdbcTemplate.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/aop
       https://www.springframework.org/schema/aop/spring-aop.xsd
       http://www.springframework.org/schema/tx
       https://www.springframework.org/schema/tx/spring-tx.xsd
">
    <context:component-scan base-package="com.mashibing"></context:component-scan>
    <context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="driverClassName" value="${jdbc.driverClassName}"></property>
    </bean>
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
    </bean>
    <!--事务控制-->
    <!--配置事务管理器的bean-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <!--开启基于注解的事务控制模式，依赖tx名称空间-->
    <tx:annotation-driven transaction-manager="transactionManager"></tx:annotation-driven>
</beans>
```


添加注解
```java
@Service
public class AService {

    @Autowired
    ADao aDao;

    @Transactional
    public void aMethod(...){
		aDao.select(...);
		aDao.update(...);
		aDao.insert(...);
		aDao.delete(...);
		...
    }
}
```

# 配置事务属性

[事务详解](https://kekaiyuan.github.io//2021/07/02/transaction/#%E4%BA%8B%E5%8A%A1)

## 传播特性
当某个事务**调用**了另一个事务，此时它们之间的关系是怎样的？<br>
传播特性就是用于处理这个的。

|  传播属性   |  描述   |
| :---: | :--- |
| REQUIRED |  如果有事务在运行，当前的方法就在这个事务内运行，否则，就启动一个新的事务，并在自己的事务内运行   |
| REQUIRES_NEW |     |
| SUPPORTS |     |
| NOT_SUPPORTED |     |
| MANDATORY |     |
| NEVER |     |
| NESTED |     |


```java
@Service
public class AService {

    @Autowired
    ADao aDao;

	@Transactional(propagation = Propagation.XXX, rollbackFor = {Exception.class})
    public void methodA() throws Exception {
		aDao.update();
		aDao.update();
    }
	
	@Transactional(propagation = Propagation.XXX, rollbackFor = {Exception.class})
    public void methodB() throws Exception{
		aDao.update();
		aDao.update();
    }
}
```

```java
@Service
public class MulService {

    @Autowired
    private AService aService;

    @Transactional
    public void mul() throws Exception {
        aService.methodA();
        aService.methodB();
    }
	
}
```
有异常都回滚


REQUIRES_NEW





### 详解 REQUIRED, REQUIRES_NEW, NESTED
这三个传播特性的描述很类似，它们具体有什么不同呢？


----------


情况一

```java
@Service
public class AService {

    @Autowired
    ADao aDao;

	@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = {Exception.class})
    public void methodA() {
		aDao.update();
		aDao.update();
        throw new Exception();
    }
	
	@Transactional
    public void methodB() {
		aDao.update();
		aDao.update();
    }
}
```

```java
@Service
public class MulService {

    @Autowired
    private AService aService;

    @Transactional
    public void mul() {
		try{
			aService.methodA();
		}catch (Exception e){
            e.printStackTrace();
		}
        aService.methodB();
    }
	
}
```
`methodA()` 抛出的异常被**捕获**了，`methodB()` 会回滚吗？<br>
**会**！<br>
如果 `methodA()` 的传播级别为 **REQUIRES_NEW** 和 **NESTED**，`methodB()` **不会回滚**！


----------






## 隔离级别
可设置事务的 [隔离级别](https://kekaiyuan.github.io//2021/07/02/transaction/#%E9%9A%94%E7%A6%BB%E6%80%A7-isolation)

使用所连接数据库**默认**的隔离级别
```java
@Transactional(isolation = Isolation.DEFAULT)
```

读未提交
```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
```

读已提交
```java
@Transactional(isolation = Isolation.READ_COMMITTED)
```

可重复读
```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

序列化
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
```

## 设置超时
可通过设置 `timeout` 属性来设置超时，单位为**秒**。<br>
当事务执行时间**超时**后，将自动**终止**该事务并**回滚**。

```java
@Transactional(timeout = 3)
```
如果该事务执行时间超过 **3s**，**自动失败并回滚**。

## 只读事务
可通过设置 `readonly` 属性来设置是否是只读事务，**默认否**。

- 执行一条查询语句<br>
	不需要开启事务，数据库默认支持 SQL 执行期间的读一致性。
- 执行多条查询语句<br>
	此时必须开启事务，保证整体的读一致性。<br>
	
当事务内只有**读**操作时，可设置该事务为只读事务。<br>
此时数据库将**阻止**其他事务修改数据。<br>
并且因为是只读事务，不存在数据修改，数据库会提供一些优化手段来**加快**读取速度。
```java
@Transactional(readOnly = true)
```

## 异常处理
当**事务**执行过程中发生异常时，**默认**情况下
- RunTimeExpcetion 及其子类会自动回滚
- 其他异常不会自动回滚

除此之外，我们可以手动配置处理异常时的回滚问题

- 声明哪些异常**不需要回滚**
	- `noRollbackFor`<br>
		填写 class 对象，使用 {}
		```java
		@Transactional(noRollbackFor = {FileNotFoundException.class})
		```
	- `noRollbackForClassName`<br>
		填写完整类名，使用 “”
		```java
		@Transactional(noRollbackForClassName = "java.io.FileNotFoundException")
		```
- 声明哪些异常**需要回滚**
	- `rollbackFor`<br>
		填写 class 对象，使用 {}
		```java
		@Transactional(rollbackFor = {FileNotFoundException.class})
		```
	- `rollbackForClassName`<br>
		填写完整类名，使用 “”
		```java
		@Transactional(rollbackForClassName = "java.io.FileNotFoundException")
		```


##### 9、事务的传播特性

事务的传播特性指的是当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行？

spring的事务传播行为一共有7种：

![传播特性](..\..\spring\image\传播特性.jpg)

##### 10、测试事务的传播特性

BookDao.java

```java
package com.mashibing.dao;import org.springframework.beans.factory.annotation.Autowired;import org.springframework.jdbc.core.JdbcTemplate;import org.springframework.stereotype.Repository;@Repositorypublic class BookDao {    @Autowired    JdbcTemplate jdbcTemplate;    /**     * 减去某个用户的余额     * @param userName     * @param price     */    public void updateBalance(String userName,int price){        String sql = "update account set balance=balance-? where username=?";        jdbcTemplate.update(sql,price,userName);    }    /**     * 按照图书的id来获取图书的价格     * @param id     * @return     */    public int getPrice(int id){        String sql = "select price from book where id=?";        return jdbcTemplate.queryForObject(sql,Integer.class,id);    }    /**     * 减库存，减去某本书的库存     * @param id     */    public void updateStock(int id){        String sql = "update book_stock set stock=stock-1 where id=?";        jdbcTemplate.update(sql,id);    }    /**     * 修改图书价格     * @param id     * @param price     */    public void updatePrice(int id,int price){        String sql = "update book set price=? where id =?";        jdbcTemplate.update(sql,price,id);    }}
```

BookService.java

```java
package com.mashibing.service;        import org.springframework.beans.factory.annotation.Autowired;        import org.springframework.stereotype.Service;        import org.springframework.transaction.annotation.Isolation;        import org.springframework.transaction.annotation.Propagation;        import org.springframework.transaction.annotation.Transactional;        import java.io.File;        import java.io.FileInputStream;        import java.io.FileNotFoundException;@Servicepublic class BookService {    @Autowired    BookDao bookDao;    /**     * 结账：传入哪个用户买了哪本书     * @param username     * @param id     */    @Transactional(propagation = Propagation.REQUIRED)    public void checkout(String username,int id) {        bookDao.updateStock(id);        int price = bookDao.getPrice(id);        bookDao.updateBalance(username,price);    }    @Transactional(propagation = Propagation.REQUIRED)    public void updatePrice(int id,int price){        bookDao.updatePrice(id,price);        int i = 1/0;    }}
```

MulService.java

```java
package com.mashibing.service;import org.springframework.beans.factory.annotation.Autowired;import org.springframework.stereotype.Service;import org.springframework.transaction.annotation.Transactional;@Servicepublic class MulService {    @Autowired    private BookService bookService;    @Transactional    public void mulTx(){        bookService.checkout("zhangsan",1);        bookService.updatePrice(1,1000);    }}
```

MyTest.java

```java
import com.mashibing.service.MulService;import org.springframework.context.ApplicationContext;import org.springframework.context.support.ClassPathXmlApplicationContext;public class MyTest {    public static void main(String[] args) {        ApplicationContext context = new ClassPathXmlApplicationContext("jdbcTemplate.xml");        MulService mulService = context.getBean("mulService", MulService.class);        mulService.mulTx();    }}
```

通过上图的结果发现，如果设置的传播特性是Required，那么所有的事务都会统一成一个事务，一旦发生错误，所有的数据都要进行回滚。

------

BookService.java

```java
package com.mashibing.service;        import com.mashibing.dao.BookDao;        import org.springframework.beans.factory.annotation.Autowired;        import org.springframework.stereotype.Service;        import org.springframework.transaction.annotation.Isolation;        import org.springframework.transaction.annotation.Propagation;        import org.springframework.transaction.annotation.Transactional;        import java.io.File;        import java.io.FileInputStream;        import java.io.FileNotFoundException;@Servicepublic class BookService {    @Autowired    BookDao bookDao;    /**     * 结账：传入哪个用户买了哪本书     * @param username     * @param id     */    @Transactional(propagation = Propagation.REQUIRES_NEW)    public void checkout(String username,int id) {        bookDao.updateStock(id);        int price = bookDao.getPrice(id);        bookDao.updateBalance(username,price);    }    @Transactional(propagation = Propagation.REQUIRED)    public void updatePrice(int id,int price){        bookDao.updatePrice(id,price);        int i = 1/0;    }}
```

通过修改checkout方法的传播特性为Required_new,发现价格进行了回滚，而其他的数据没有进行回滚。

------

BookService.java

```java
package com.mashibing.service;        import com.mashibing.dao.BookDao;        import org.springframework.beans.factory.annotation.Autowired;        import org.springframework.stereotype.Service;        import org.springframework.transaction.annotation.Isolation;        import org.springframework.transaction.annotation.Propagation;        import org.springframework.transaction.annotation.Transactional;        import java.io.File;        import java.io.FileInputStream;        import java.io.FileNotFoundException;@Servicepublic class BookService {    @Autowired    BookDao bookDao;    /**     * 结账：传入哪个用户买了哪本书     * @param username     * @param id     */    @Transactional(propagation = Propagation.REQUIRED)    public void checkout(String username,int id) {        bookDao.updateStock(id);        int price = bookDao.getPrice(id);        bookDao.updateBalance(username,price);    }    @Transactional(propagation = Propagation.REQUIRED)    public void updatePrice(int id,int price){        bookDao.updatePrice(id,price);    }}
```

MulService.java

```java
package com.mashibing.service;import com.mashibing.bean.Book;import org.springframework.beans.factory.annotation.Autowired;import org.springframework.stereotype.Service;import org.springframework.transaction.annotation.Transactional;@Servicepublic class MulService {    @Autowired    private BookService bookService;    @Transactional    public void mulTx(){        bookService.checkout("zhangsan",1);        bookService.updatePrice(1,1000);        int i = 1/0;    }}
```

将bookservice方法的传播行为为Required，并且将报错设置在MulService中，发现会都进行回滚。

------

BookService.java

```java
package com.mashibing.service;        import com.mashibing.dao.BookDao;        import org.springframework.beans.factory.annotation.Autowired;        import org.springframework.stereotype.Service;        import org.springframework.transaction.annotation.Isolation;        import org.springframework.transaction.annotation.Propagation;        import org.springframework.transaction.annotation.Transactional;        import java.io.File;        import java.io.FileInputStream;        import java.io.FileNotFoundException;@Servicepublic class BookService {    @Autowired    BookDao bookDao;    /**     * 结账：传入哪个用户买了哪本书     * @param username     * @param id     */    @Transactional(propagation = Propagation.REQUIRES_NEW)    public void checkout(String username,int id) {        bookDao.updateStock(id);        int price = bookDao.getPrice(id);        bookDao.updateBalance(username,price);    }    @Transactional(propagation = Propagation.REQUIRES_NEW)    public void updatePrice(int id,int price){        bookDao.updatePrice(id,price);    }}
```

MulService.java

```java
package com.mashibing.service;import com.mashibing.bean.Book;import org.springframework.beans.factory.annotation.Autowired;import org.springframework.stereotype.Service;import org.springframework.transaction.annotation.Transactional;@Servicepublic class MulService {    @Autowired    private BookService bookService;    @Transactional    public void mulTx(){        bookService.checkout("zhangsan",1);        bookService.updatePrice(1,1000);        int i = 1/0;    }}
```

将bookservice方法的传播行为为Requires_new，并且将报错设置在MulService中，发现都不会进行回滚。

------

BookService.java

```java
package com.mashibing.service;        import com.mashibing.dao.BookDao;        import org.springframework.beans.factory.annotation.Autowired;        import org.springframework.stereotype.Service;        import org.springframework.transaction.annotation.Isolation;        import org.springframework.transaction.annotation.Propagation;        import org.springframework.transaction.annotation.Transactional;        import java.io.File;        import java.io.FileInputStream;        import java.io.FileNotFoundException;@Servicepublic class BookService {    @Autowired    BookDao bookDao;    /**     * 结账：传入哪个用户买了哪本书     * @param username     * @param id     */    @Transactional(propagation = Propagation.REQUIRES_NEW)    public void checkout(String username,int id) {        bookDao.updateStock(id);        int price = bookDao.getPrice(id);        bookDao.updateBalance(username,price);    }    @Transactional(propagation = Propagation.REQUIRES_NEW)    public void updatePrice(int id,int price){        bookDao.updatePrice(id,price);    }    @Transactional    public void mulTx(){        checkout("zhangsan",1);        updatePrice(1,1000);        int i = 1/0;    }}
```

如果在bookservice执行的话，会发现刚刚的效果就没有了，原因是外层调用的时候使用的AOP，但是本类方法自己的调用就是最最普通的调用，就是同一个事务。

总结：

```tex
1、事务传播级别是REQUIRED，当checkout()被调用时（假定被另一类中commit()调用），如果checkout()中的代码抛出异常，即便被捕获，commit()中的其他代码都会roll back2、是REQUIRES_NEW，如果checkout()中的代码抛出异常，并且被捕获，commit()中的其他代码不会roll back；如果commit()中的其他代码抛出异常，而且没有捕获，不会导致checkout()回滚3、是NESTED，如果checkout()中的代码抛出异常，并且被捕获，commit()中的其他代码不会roll back；如果commit()中的其他代码抛出异常，而且没有捕获，会导致checkout()回滚    PROPAGATION_REQUIRES_NEW 启动一个新的, 不依赖于环境的 "内部" 事务. 这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行.     另一方面, PROPAGATION_NESTED 开始一个 "嵌套的" 事务,  它是已经存在事务的一个真正的子事务. 嵌套事务开始执行时,  它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 潜套事务是外部事务的一部分, 只有外部事务结束后它才会被提交.     由此可见, PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED 的最大区别在于, PROPAGATION_REQUIRES_NEW 完全是一个新的事务, 而 PROPAGATION_NESTED 则是外部事务的子事务, 如果外部事务 commit, 嵌套事务也会被 commit, 这个规则同样适用于 roll back. 
```



### 3、基于xml的事务配置

jdbcTemplate.xml

```xml
<?xml version="1.0" encoding="UTF-8"?><beans xmlns="http://www.springframework.org/schema/beans"       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"       xmlns:context="http://www.springframework.org/schema/context"       xmlns:aop="http://www.springframework.org/schema/aop"       xmlns:tx="http://www.springframework.org/schema/tx"       xsi:schemaLocation="http://www.springframework.org/schema/beans       http://www.springframework.org/schema/beans/spring-beans.xsd       http://www.springframework.org/schema/context       http://www.springframework.org/schema/context/spring-context.xsd       http://www.springframework.org/schema/aop       https://www.springframework.org/schema/aop/spring-aop.xsd       http://www.springframework.org/schema/tx       https://www.springframework.org/schema/tx/spring-tx.xsd">    <context:component-scan base-package="com.mashibing"></context:component-scan>    <context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">        <property name="username" value="${jdbc.username}"></property>        <property name="password" value="${jdbc.password}"></property>        <property name="url" value="${jdbc.url}"></property>        <property name="driverClassName" value="${jdbc.driverClassName}"></property>    </bean>    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">        <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>    </bean>    <bean id="namedParameterJdbcTemplate" class="org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate">        <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>    </bean>    <!--事务控制-->    <!--配置事务管理器的bean-->    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">        <property name="dataSource" ref="dataSource"></property>    </bean>    <!--    基于xml配置的事务：依赖tx名称空间和aop名称空间        1、spring中提供事务管理器（切面），配置这个事务管理器        2、配置出事务方法        3、告诉spring哪些方法是事务方法（事务切面按照我们的切入点表达式去切入事务方法）    -->    <bean id="bookService" class="com.mashibing.service.BookService"></bean>    <aop:config>        <aop:pointcut id="txPoint" expression="execution(* com.mashibing.service.*.*(..))"/>        <!--事务建议：advice-ref:指向事务管理器的配置-->        <aop:advisor advice-ref="myAdvice" pointcut-ref="txPoint"></aop:advisor>    </aop:config>    <tx:advice id="myAdvice" transaction-manager="transactionManager">        <!--事务属性-->        <tx:attributes>            <!--指明哪些方法是事务方法-->            <tx:method name="*"/>            <tx:method name="checkout" propagation="REQUIRED"/>            <tx:method name="get*" read-only="true"></tx:method>        </tx:attributes>    </tx:advice></beans>
```



# 源码链接
该文章源码链接 [Github](url)