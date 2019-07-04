# 1、事务基础知识
## 1、1 事务定义
+ 数据库管理系统执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成
+ 事务的核心就是锁与并发控制
## 1、2 数据库隔离级别
+ 未提交读：事务中的修改即使没有提交也对其他事务可见。
+ 提交读：事务内只能看到其他已经提交的事务结果。
+ 可重复读：只允许读取已经提交的数据，而且在一个事务两次读取一个数据项期间，其它事务不得更新该数据。
+ 序列读：最高的隔离级别，事务串行执行

## 1、3 事务总结
## 事务单元操作类型
+ 读读
+ 读写
+ 写读
+ 写写

## 事务并发问题
+ 更新丢失：当两个不同的事务试图同时更新数据库中同一行上的同一列时，会发生丢失更新
+ 脏读：即事务可以读取未提交的数据，写与读不互斥。
+ 不可重复读：即同一个事务多次查询得到不一致的结果，读写不互斥
+ 幻读：当某个事物在读取某个范围内的记录时，另外一个事务又在该范围内插入了新的记录，当之前的事务再次读取该范围的记录时，会产生幻行


## 事务特性关系
+ 隔离性与一致性：是一致性与并发之间的杠杆，隔离级别增加，一致性增加，并发性能降低，隔离线打破了一致性。
+ 原子性与一致性：两者没有必然联系，原子性保证的事务操作要么全做、要么全不做，对每个操作按时间做记录，以便回滚操作，一致性保持的数据状态变更，中间状态对外部不可见。

## 事务模型
+ 本地事务模型
    + （面向连接，不是由框架或者容器进行管理）
+ 编程事务模型
    + （从事务管理器中获取事务，之后需要自己编写事务启动、提交、异常及回滚代码）
+ 声明事务模型
    + （容器管理事务，开发者定义（声明）事务的行为和参数）



# 2、Mybatis
## 2、1 整体架构
![](../images/spring/nybatis架构.jpg)

## 2、2 项目结构
 ```java
.
└── org
    └── apache
        └── ibatis
            ├── annotations
            ├── binding
            ├── builder
            ├── cache
            ├── cursor
            ├── datasource
            ├── exceptions
            ├── executor
            ├── io
            ├── jdbc
            ├── logging
            ├── mapping
            ├── parsing
            ├── plugin
            ├── reflection
            ├── scripting
            ├── session
            ├── transaction
            └── type
```

## 2、3 一级缓存配置介绍

```java
    <settings>
        <setting name="localCacheScope" value="SESSION"/>//一级缓存级别为SESSION、STEMENT 
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        <setting name="logImpl" value="LOG4J"/>
    </settings>
  
``` 

## 2、4 一级缓存工作流程与原理
### 2、4、1 一级缓存类图
+ ![](../images/spring/Mybatis.bmp)

+ SqlSession 
    + 1）mybatis中主要接口，该接口主要负责执行SQL并封装返回结果，可以获取mapper、管理事务；
    + 2）对外提供了用户和数据库之间交互需要的所有方法，隐藏了底层的细节，默认实现类是DefaultSqlSession
+ Executor
    + 1）每个Session都会持有一个执行器，数据库的CRUD操作都是通过执行器进行
    + 2）该接口有两个实现类一个是BaseExecutor、CachingExecutor;其中BaseExcutor有三个继承类SimpleExecutor（普通执行器）、ReuseExecutor（重用预处理执行器）、BatchExecutor（重用语句批量执行器）
    + CachingExecutor来装饰前面的三个执行器目的就是用来实现缓存
+ MappedStatement
     + MappedStatement就是用来存放我们SQL映射文件中的信息包括sql语句，输入参数，输出参数等等，一个SQL节点对应一个MappedStatement对象。
### 2、4、2 案例实践
### 案例1
```java
    SqlSession sqlSession = factory.openSession(true); // 自动提交事务
    StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);

    System.out.println(studentMapper.getStudentById(1));
    System.out.println(studentMapper.getStudentById(1));
    System.out.println(studentMapper.getStudentById(1));

    sqlSession.close();

``` 

```java
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 小明, 16
    DEBUG [main] - <==      Total: 1
    StudentEntity{id=1, name='小明', age=16, className='null'}
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    StudentEntity{id=1, name='小明', age=16, className='null'}
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    StudentEntity{id=1, name='小明', age=16, className='null'}

``` 
+ 同一个Session，第一个打印数据查询数据库，第二、三个查询走缓存
+ 思考自动提交事务底层实现原理？

#### 案例2
```java
    SqlSession sqlSession = factory.openSession(true); // 自动提交事务
    StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
    System.out.println(studentMapper.getStudentById(1));
    System.out.println("增加了" + studentMapper.addStudent(buildStudent()) + "个学生");
    System.out.println(studentMapper.getStudentById(1));

    sqlSession.close();
```
```java
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 小明, 16
    DEBUG [main] - <==      Total: 1
    StudentEntity{id=1, name='小明', age=16, className='null'}
    DEBUG [main] - ==>  Preparing: INSERT INTO student(name,age) VALUES(?, ?) 
    DEBUG [main] - ==> Parameters: 张三疯(String), 20(Integer)
    DEBUG [main] - <==    Updates: 1
    增加了1个学生
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 小明, 16
    DEBUG [main] - <==      Total: 1
    StudentEntity{id=1, name='小明', age=16, className='null'}
``` 

+ 同一个Session，第一个查询走数据库
+ 先做增加再做查询，直接查询数据库
+ 思考有insert会清零缓存，如何清理？

#### 案例3
```java
    SqlSession sqlSession1 = factory.openSession(true); // 自动提交事务
    SqlSession sqlSession2 = factory.openSession(true); // 自动提交事务

    StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
    StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);

    System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
    System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
    System.out.println("studentMapper2更新了" + studentMapper2.updateStudentName("李四",1) + "个学生的数据");
    System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
    System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));

``` 

```java
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 小明, 16
    DEBUG [main] - <==      Total: 1
    studentMapper读取数据: StudentEntity{id=1, name='小明', age=16, className='null'}
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    studentMapper读取数据: StudentEntity{id=1, name='小明', age=16, className='null'}
    DEBUG [main] - ==>  Preparing: UPDATE student SET name = ? WHERE id = ? 
    DEBUG [main] - ==> Parameters: 李四(String), 1(Integer)
    DEBUG [main] - <==    Updates: 1
    studentMapper2更新了1个学生的数据
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    studentMapper读取数据: StudentEntity{id=1, name='小明', age=16, className='null'}
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 李四, 16
    DEBUG [main] - <==      Total: 1
    studentMapper2读取数据: StudentEntity{id=1, name='李四', age=16, className='null'}

``` 
+  分别创建两个Session
+  Session1 第一次查询数据直接走数据库，第二次查询数据走缓存
+  Session2 更新数据，更新ID为1的学生的姓名
+  Session1 第三次读取数据，读取的数据还是缓存的数据
+  Session2 查询数据，读取的是更新过姓名的数据
+  缓存在Session隔离，相互不影响，此案例出现脏数据

### 2、4、3一级缓存工作流程
+ ![](../images/spring/Mybatis一级缓存.bmp)


## 2、5  一级缓存总结
+ 1、一级缓存支持两种级别，分别是SESSION、STATEMENT；STATEMENT相当于不使用一级缓存，默认是SESSION级别；
+ 2、一级缓存设计比较简单，针对缓存的功能比较弱，只是简单的用Map来存储，它的生命周期同SQL Session
## 2、6  二级缓存配置介绍

```java
    <settings>
        <setting name="localCacheScope" value="SESSION"/>//一级缓存级别为SESSION、STEMENT
        <setting name="cacheEnabled" value="true"/>//二级缓存
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        <setting name="logImpl" value="LOG4J"/>
    </settings>
  
``` 
### 案例1
```java
    SqlSession sqlSession1 = factory.openSession(true); // 自动提交事务
    SqlSession sqlSession2 = factory.openSession(true); // 自动提交事务
    SqlSession sqlSession3 = factory.openSession(true); // 自动提交事务

    StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
    StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);
    StudentMapper studentMapper3 = sqlSession3.getMapper(StudentMapper.class);

    System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
    sqlSession1.close();

    System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));

    studentMapper3.updateStudentName("方方",1);
    sqlSession3.commit();
    System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));

```
执行结果
```java
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.0
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 方方, 16
    DEBUG [main] - <==      Total: 1
    studentMapper读取数据: StudentEntity{id=1, name='方方', age=16, className='null'}
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.5
    studentMapper2读取数据: StudentEntity{id=1, name='方方', age=16, className='null'}
    DEBUG [main] - ==>  Preparing: UPDATE student SET name = ? WHERE id = ? 
    DEBUG [main] - ==> Parameters: 方方(String), 1(Integer)
    DEBUG [main] - <==    Updates: 1
    DEBUG [main] - Cache Hit Ratio [com.jd.jr.finetch.study.mybatis.mapper.StudentMapper]: 0.3333333333333333
    DEBUG [main] - ==>  Preparing: SELECT id,name,age FROM student WHERE id = ? 
    DEBUG [main] - ==> Parameters: 1(Integer)
    TRACE [main] - <==    Columns: id, name, age
    TRACE [main] - <==        Row: 1, 方方, 16
    DEBUG [main] - <==      Total: 1
    studentMapper2读取数据: StudentEntity{id=1, name='方方', age=16, className='null'}
```
+ 分别创建三个Session
+ Session1查询学生Mapper ID为1的数据，首次查询所以走数据库，并且查完数据手动关闭连接
+ Session2查询学生Mapper ID为1的数据，首次查询但是可以看出没有走数据库，为什么？Session隔离的


## 2、7  二级缓存工作流程与原理
## 2、8  二级缓存总结

# 3、Spring事务
## 3、1 Spring事务管理接口介绍
## 3、2 Spring声明事务案例与最佳实践
## 3、3 Spring事务执行原理







# 参考资料
+ http://www.mybatis.org/mybatis-3/zh
+ https://docs.spring.io/spring/docs/3.0.x/spring-framework-reference/html/transaction.html
+ https://github.com/mybatis/mybatis-3
