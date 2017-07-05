---
title: MyBatis最小系统搭建
date: 2017-05-26 10:43:08
tags: {JavaWeb, Spring}
---

有一句老话叫做:

> 如果没有好好看官方文档，你是不会有好结果的。

不管这句话的出处是什么，重要的是不要从二手资料中去寻找答案，比如你现在正在看的这篇文章。这两天花了一些时间了解了MyBatis，并整合和Spring，过程不复杂但是不可避免踩了一些坑。

其实如果了解了MyBatis的机制，那么与Spring的整合就显得水到渠成了。首先，我们从搭建一个纯MyBatis环境开始。

# MyBatis环境搭建

原材料：intellij idea+maven+mysql

新建一个maven的工程，在pom.xml中加入如下依赖包：

```xml
<dependencies>
      <dependency>
          <groupId>org.mybatis</groupId>
          <artifactId>mybatis</artifactId>
          <version>3.4.4</version>
      </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>

      <dependency>
          <groupId>mysql</groupId>
          <artifactId>mysql-connector-java</artifactId>
          <version>5.1.39</version>
      </dependency>
  </dependencies>
<!--maven默认寻找资源文件是在main/resources目录下面，而一般情况下mapper文件放在类的同级别目录下面，因此需要在pom.xml中告诉maven搜寻资源文件的位置，否则会出现could not find resource报错。-->
 <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <includes>
                    <include>**/*.properties</include>
                    <include>**/*.xml</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.properties</include>
                    <include>**/*.xml</include>
                </includes>
            </resource>
        </resources>
    </build>
```

强烈建议使用maven管理项目的包依赖（如果你和我一样曾经被包依赖搞晕头）。

MyBatis使用SqlSessionFactory作为每一个应用的核心，SqlSessionFactory由SqlSessionFactoryBuilder通过XML配置或者Configuration实例构建出来的。

```java
mInputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

从sqlSessionFactory中获取SqlSession，SqlSession可以获得执行已经映射过的SQL语句。

```java
SqlSession session = sqlSessionFactory.openSession();
```

我们看一下完整测试程序：

```java
public class myBatisSimpleTest {
    @Test
    public void myBatisSimpleTest() throws Exception{
        String resource = "mybatis-config.xml";//指出配置文件的位置
        InputStream inputStream = Resources.getResourceAsStream(resource);//读取配置文件
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);//由配置文件得到核心的SqlSessionFactoryBuilder
        SqlSession session = sqlSessionFactory.openSession();//开启一个事务，这里注意一下和Hibernate不同，Hibernate可以用getCurrentSession获取，并且自动close，mybatis是手动关闭的。
        try {
            IUserInfoDAO userInfoDAO = session.getMapper(IUserInfoDAO.class);//实例化
            UserInfo userInfo = userInfoDAO.selectUser("tangbin");
            System.out.println(userInfo.toString());
        } finally {
            session.close();//关闭事务
        }
    }
}
```

这里应该会有个疑问：mybatis-config.xml是啥。

## MyBatis全局配置文件

mybatis-config.xml是MyBatis的核心配置，用于获取数据库连接、事务作用域等等。设置mybatis-config.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="jdbc.properties"/>
    <environments default="development">
        <environment id="development">
          <!--采用JDBC提供的提交和回滚机制，而JDBC依赖数据源得到的连接来管理事务的作用域。-->
            <transactionManager type="JDBC"/>
          <!--这种数据源的实现利用“池”的概念将 JDBC 连接对象组织起来，避免了创建新的连接实例时所必需的初始化和认证时间。 这是一种使得并发 Web 应用快速响应请求的流行处理方式。-->
            <dataSource type="POOLED">
                <property name="driver" value="${driverClass}"/>
                <property name="url" value="${jdbcUrl}"/>
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="org/mybatis/example/dao/userinfodao.xml"/><!--这里我们要指定一个接口的映射文件，当然如果是注解的方式就没有必要了-->
        <!--<mapper class="com.tangbin.dao.IUserDao"/>-->
        <!--<package name="com.tangbin.dao.IUserDao"/>-->
    </mappers>

</configuration>
```

jdbc.properties必须在classpath能够找到的目录下面。最后看一下mappers，告诉了MyBatis映射文件的位置。mybatis-config.xml是MyBatis的SqlSessionFactory整体配置文件，接下来我们要为每一个接口中的每个方法做sql语句映射。

## XML映射文件

首先我们要声明一个接口：

```java
package org.mybatis.example.dao;
import org.mybatis.example.entity.UserInfo;

public interface IUserInfoDAO {
    UserInfo selectUser(String username);
}
```

其对应的映射文件如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="org.mybatis.example.dao.IUserInfoDAO">
    <select id="selectUser" parameterType="String" resultType="org.mybatis.example.entity.UserInfo"><!--此处要注意where语句后的#部分要和接口参数名称一致，resultType为全名-->
        SELECT * from USER WHERE name=#{username}
    </select>
</mapper>
```

其中username是我们传入查询条件。如果传入一个对象则同样需要指定相关对象类型：

```xml
<insert id="insertUser" parameterType="User">
  insert into users (id, username, password)
  values (#{id}, #{username}, #{password})
</insert>
```

MyBaits的映射文件其实还有很多东西可以深入，比如一对多的问题等等，这里不再多说，毕竟我们的目标是建立一个最小的能够顺利运行的MyBatis框架。

# 总结

到这里一套MyBatis环境就搭建完毕了，下面我们用一张图来总结一下。

<img src="http://7xp3xc.com1.z0.glb.clouddn.com/201601/1499186581824.png" width="573"/>

Mybatis有四个要素，这四个要素都是层层依赖的关系：

* SqlSessionFactory是每个Mybatis的核心，它用来构造每一个Mybatis的Session并操作Sql。它依赖于MyBatis-Config.xml
* MyBatis-Config.xml是MyBatis的全局配置文件，包含数据库的设置，并指出映射文件(XXXMaper.xml)的具体位置，用来构造SqlSessionFactory。
* 映射文件XXXMapper.xml用来做接口每一个方法的映射，依赖于接口的定义。
* Interface定义我们要对数据库进行的操作。

Mybatis框架的搭建就在这里告一段落，下一篇文章我们尝试将MyBatis与Spring结合起来。