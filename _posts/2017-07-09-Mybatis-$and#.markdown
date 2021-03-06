---
layout: post
title: "关于Mybatis的$和#,你真的知道他们的细节吗?"
date: 2017-07-09
tags:
    - java
    - mybatis
    - sql注入
    - 17年7月
keywords: mybatis
description: 关于Mybatis的$和#,你真的知道他们的细节吗?
---

### 前言

在JDBC中，主要使用的是两种语句，一种是支持参数化和预编译的PrepareStatement，能够支持原生的Sql，也支持设置占位符的方式，参数化输入的参数，防止Sql注入，一种是支持原生Sql的Statement，有Sql注入的风险。

在使用Mybatis进行开发过程中，隐藏了底层具体使用哪一种语句的细节，我们通过使用#和$告诉Mybatis，我们实际上进行的是怎么样的操作，需要对语句进行参数化还是说直接保持原生状态就好。

今天我们主要看一下使用两种符号使用时系统应对Sql注入的表现和Mybatis在内部是如何对他们处理的源码分析。

### #和$在应对Sql注入上的区别表现

> 利用现有应用程序，将（恶意的）SQL命令注入到后台数据库引擎执行的能力，它可以通过在Web表单中输入（恶意）SQL语句得到一个存在安全漏洞的网站上的数据库，而不是按照设计者意图去执行SQL语句。

比如说根据学生姓名查学生信息，会传入一个name的参数，假设学生姓名是方方，那么Sql就是

```
SELECT id,name,age FROM student WHERE name = '方方'; 
```

在没有做防Sql注入的时候，我们的Sql语句可能是这么写的

```
<select id="fetchStudentByName" parameterType="String" resultType="entity.StudentEntity"> SELECT id,name,age FROM student WHERE name = '${value}' </select> 
```

正常情况下查出姓名符合方方的学生信息。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-10-06-45.jpg)

但如果我们对传入的姓名参数做一些更改，比如改成anything' OR 'x'='x，那么拼接而成的Sql就变成了

```
SELECT id,name,age FROM student WHERE name = 'anything' OR 'x'='x' 
```

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-10-08-30.jpg)

库里面所有的学生信息都被拉了出来，是不是很可怕。原因就是传入的anything' OR 'x'='x和原有的单引号，正好组成了 'anything' OR 'x'='x'，而OR后面恒等于1，所以等于对这个库执行了查所有的操作。

**防范Sql注入的话，就是要把整个anything' OR 'x'='x中的单引号作为参数的一部分，而不是和Sql中的单引号进行拼接**

使用了$即可在Mybatis中对参数进行转义

```
<select id="fetchStudentByName" parameterType="String" resultType="entity.StudentEntity"> SELECT id,name,age FROM student WHERE name = #{name} </select> 
```

我们看一下发送到数据库端的Sql语句长什么样子。

```
SELECT id,name,age FROM student WHERE name = 'anything\' OR \'x\'=\'x' 
```

从上述代码中我们可以看到参数中的所有单引号统统被转移了，这都是JDBC中PrepareStatement的功劳，如果在数据库服务端开启了预编译，则是服务端来做了这件事情。

具体可以看我之前写的这篇: JDBC与Mysql的那些事,里面解释了为何PrepareStatement能做到这件事情。

### 源码

在以前的文章中，我们说明过Mybatis的执行流程主要部件，SqlSession 提供给用户操作的Api，Executor 具体执行对数据库的操作，但其实在Executor内部还会再委托给StatementHandler这个接口。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-08-42-01.jpg)

这个Handler的实现类就是代表了JDBC中的操作语句,CallableStatementHandler、PrepareStatementHandler和SimpleStatementHandler就会代表对JDBC中的CallableStatement，PrepareStatement和Statement，这些handler的内部就会调用JDBC中的相关Statement。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-08-44-17.jpg)

类比Mybatis的执行流程和JDBC原有的我们使用的方法就是。

Mybatis: Sqlsession -> Executor -> StatementHandler -> ResultHandler

JDBC: Connection -> Statement -> Result

因此我们可以知道对JDBC语句的操作都会在StatementHandler内部。

在PrepareStatementHandler中会使用paramterize对Statement进行参数化，在其中他会委托给DefualtParameterHandler进行操作。我们通过两种不同的语句，看一下，Debug下这段代码的不同。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-08-54-08.jpg)

首先是使用$符号，它是会直接在Sql中进行拼接的，从下图可知，在进行参数化的时候，Sql语句已经被拼接完成了，见originSql。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-08-58-15.jpg)

进入DefualtParameterHandler内部，如下图可知，我们看到，这儿boundSql的ParameterMappings不存在，所以不用执行第二个红框处，设置对应占位符的操作。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-08-59-42.jpg)

然后，我们看一下当使用#的时候，同样的代码，会得到什么样的处理结果。从下图可知，当使用#的时候，原有的#{value}被替换成了？号，也就是我们熟知的JDBC中的占位符。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-02-37.jpg)

再进入DefualtParameterHandler的时候， 此时会有ParameterMappings，value -> anything' OR 'x'='x'，找到合适的TypeHandler塞入PrepareStatement中。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-04-48.png)

**从上文的分析中，我们得到的就是，当使用的时候，的时候，{value}，是直接被替换为了对应的值，没有参数映射，不会进行设置占位符的操作，当使用#的时候，#{}会被替换为?号，有参数映射,会在DefaultParameterHandler中进行设置占位符的操作。

**问题**

1 为什么默认使用的语句是PrepareStatementHandler

2 和#是什么时候被替换的，为什么对应的BoundSql，$时没有映射，#有映射。

带着这两个问题我们来看一下，Mybatis的初始化阶段，为节省篇幅，仅列出大致路径，和关键代码。

----

Mybatis是通过SqlSessionFactory build出来的，会解析映射文件，大致路径就是

SqlSessionFactoryBuilder -> XmlConfigBuilder->XMLMapperBuilder->XMLStatementBuilder。

在XMLStatementBuilder的parseStatementNode负责了生成MappedStatement，首先回答第一个问题。当你不指定statementType时，Mybatis默认使用的就是PrepareStatementHandler，这里的StatementType，在后续流程中使用RoutingStatementHandler选择使用哪一个StatementHandler。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-19-47.jpg)

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-21-26.jpg)

然后继续看第二个问题，$和#是怎么被替换的。

在之前我们提到了，BoundSql中包含了Sql主体，同时其中的参数映射决定了后续是否要进行参数化,在$和#时，表现是不同的。

BoudSql来自于MappedStatement，在MappedStatement中，获取BoundSql的任务会委托给SqlSource接口。所以我们接下来主要看SqlSource是如何生成的。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-24-08.jpg)

XMLLandDriver可以理解为就是用来解析Mybatis定制的XML符号的语句。他会把具体解析符号的职责交给XMLScriptBuilder的parseScriptNode方法。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-26-12.jpg)

parseDynamicTags中会把语句用TextSql包装起来，然后使用isDynamic方法，在方法中使用GerenericTokenParser判断是否是动态语句。如果其中包含$，就是动态的，如果是#就不是动态的，使用的Handler是DrynamicCheckerTokenParser。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-36-22.jpg)

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-27-45.jpg)

在进入parse方法后，主要看以下这一段。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-38-10.jpg)

这里会使用TokenHandler不同的实现类，对表达式进行进一步的处理，这里是对Sql自后的完善，在判断isDynamic中，使用的是DrynamicCheckerTokenParser，一个最简单的实现。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-39-15.jpg)

parse完成后，如果isDynamic是true的话，就是动态语句，使用DynamicSqlSource。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-31-01.jpg)

如果是非动态的话，其实一般就是指使用了#的语句，使用RawSqlSource，在其中，还会进一步解析。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-33-24.jpg)

从下图中可以看到，这个TokenParser这回使用的是#{}，而且使用的是ParameterMappingTokenHandler。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-34-12.jpg)

ParameterMappingTokenHandler的handlerToken方法中，完成了添加参数映射和替换#{value}为？的职责。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-41-00.jpg)

从以上我们可以知道，使用#在初始化阶段，会被替换成？号，同时生成参数映射，而使用$在初始化阶段，没有什么特别的地方，仅仅做了一个是否动态语句的判断。

----

在初始化完毕后，我们进入getBoundSql方法，看一下DynamicSqlSource和StaticSource在此刻做了什么，首先是DynamicSqlSource。

在其中，首先会生成一个DynamicContext,主要就是 生成bindings，一个是 "\_parameter" -> "anything' OR 'x'='x",一个是"\_databaseId" -> "null"

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-46-34.jpg)

然后使用了apply方法，我理解这里是要去做替换了。具体还是使用${}去判断，和上文一致，只不过这里使用的是BindingTokenParser。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-48-13.jpg)

看一下BindingTokenParser的HandleToken方法。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-50-32.jpg)

上述代码的效果，就是会使用Ognl,使用value在Bindings中，找对应的值，最后返回，拼接在Sql中，这也就是为什么会有Sql注入风险的原因。使用value是因为Ognl去找的时候，就会使用value这个默认值，所以需要在bindings额外加入这么一个键值对，有兴趣可以继续往下看ONGL相关的东西。

接下来是生成SqlSource，使用的是SqlSourceBuilder的parse方法。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-53-57.jpg)

在前文介绍过，在这个parse方法里，是用#{}来判断的，所以走不到ParameterMappingTokenHandler的handlerToken方法，也就无法添加参数映射了，这个直接返回一个StaticSqlSource，这也解释了为什么使用$时，参数映射为空。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-55-35.jpg)

再接下去就是获取BoundSql，使用的是StaticSqlSource，直接根据参数，实例化了一个，参数映射为空。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-57-15.jpg)

当使用#的时候，使用的就是StaticSqlSource，直接实例化，因为参数映射在之前初始化的阶段，也生成好了，所以很简单的一个流程。

![](http://or7kd5uf1.bkt.clouddn.com/%E5%86%99%E4%BD%9C/%E5%8D%9A%E6%96%87/_image/2017-07-09-09-57-15.jpg)

后续的流程，就和Mybatis正常的流程一致了。

### 总结

本文主要剖析了Mybatis中$和#两种符号使用上的不同，以及使用这两种符号时，源码流程上的区别。建议大家都使用#号，在orm这层也规避到Sql注入的风险。

![输入图片说明](https://static.oschina.net/uploads/img/201707/09103417_O58Q.jpg "在这里输入图片标题")

**倘若您有疑问或者有进一步想了解内容，欢迎留言给我。**