---
layout: post
title: "Java反射机制学习(三)-附上demo"
date: 2017-12-10
tags:
    - Java
    - 反射
keywords: Java,反射
description: Java反射机制学习(三)
---
经过前面几次反射机制的学习，这次用反射的知识写一个类似于Struts框架处理机制的小demo。

### Servlet 和 Sturts
在引入反射知识前，先简单介绍下Sturts框架和Servlet。
在没有一些Web框架之前，当我们要写Java Web应用使用的就是Servlet。一个简单的Servletdemo就是如下所示。
```java
public class HelloWorld extends HttpServlet {
 
   private String message;

   public void init() throws ServletException {
      message = "Hello World";
   }

   public void doGet(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {
      response.setContentType("text/html");
      PrintWriter out = response.getWriter();
      out.println("<h1>" + message + "</h1>");
   }

   public void destroy() {
   }
}
```
servlet会提供出来doGet和doPost，同时接收用户传入的参数，进行业务处理，再返回视图。那么Servlet如何与URL对应起来呢，答案就是在web.xml，绑定servlet和url之间的映射关系。
```xml
<servlet>
   <servlet-name>HelloWorld</servlet-name>
   <servlet-class>HelloWorld</servlet-class>
</servlet>

<servlet-mapping>
   <servlet-name>HelloWorld</servlet-name>
   <url-pattern>/HelloWorld</url-pattern>
</servlet-mapping>
```
映射、业务逻辑处理、视图返回全部在servlet中完成，耦合度比较高，随着url的增多，servlet会越来越多，需要在web.xml配置很多映射关系，不利于维护。同时servlet的入参以及返回的参数很依赖于当前运行的容器，本身也是线程不安全的，当入参非常多时，需要多次调用getParm方法，代码很冗余。
之后Struts框架诞生，通过统一的ActionServlet处理具体的url请求和参数映射以及根据不同的返回结果跳转不同的视图，开发者只需要关心自己的业务逻辑，就可以实现web应用的开发。具体的Struts的配置文件，大致如下面XML所示。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<struts>
    <action name="login" class="com.coderising.kailuncen.LoginAction">
        <result name="success">/jsp/homepage.jsp</result>
        <result name="fail">/jsp/showLogin.jsp</result>
    </action>
    <action name="logout" class="com.coderising.kailuncen.LogoutAction">
    	<result name="success">/jsp/welcome.jsp</result>
    	<result name="error">/jsp/error.jsp</result>
    </action>
</struts>
```
我们只需要分别实现视图和业务逻辑，再通过struts将他们绑定起来，就可以完成开发工作，也更便于理解，方便维护。有兴趣的读者可以自行深入学习下servlet和struts的思想。

### 小demo
我想写的小demo就是利用读取xml，利用反射加载不同的action，进行业务逻辑处理，最后输出返回的视图，整个逻辑思路还是比较简单，纯当反射学习的练手。
首先是定义配置类，把xml中action对应的映射关系保存下来
```java
private class ActionConfig {
		private String name;
		private String className;
		private Map<String, String> viewResult = new HashMap<>();
```
当初始化读取xml完毕后，得到如下结构，action的名字对应着具体的action配置
```java
Map<String, ActionConfig> actionConfigMap = new HashMap<>();
```
模拟Struts ActionServlet的运作方式
```java
public View runAction(String actionName, Map<String, String> params) {
		String className = cfg.getClassName(actionName);
		if (className == null) {
			return null;
		}
		try {
			Class<?> clz = Class.forName(className);
			Object action = clz.newInstance();
			ReflectionUtil.invokeSetMethods(action, params);
			String resultName = (String) clz.getDeclaredMethod("execute").invoke(action);
			Map<String, Object> result = ReflectionUtil.invokeGetMethods(action);
			String resultView = cfg.getViewResult(actionName, resultName);
			return new View(resultView, result);
```
通过actionName从配置类中拿到具体的执行类的全类名，其实Struts框架就是直接解析url，然后对应到xml配置的对应action名称，将url和具体的执行类绑定在一起。
之后是使用Class.forName创建类类型，然后创建对应的实例。ReflectionUtil里面做的事情就是，先获取action中对应的field的Name，然后从变量中，根据filed名称找对应的值，然后使用set方法对action的field进行赋值操作，就是LoginAction中的相关信息。
```java
public class LoginAction {
	private String name;
	private String password;
	private String message;
```
这一步就省去了使用servlet时，重复去get赋值的繁琐操作，利用反射机制，直接对成员变量进行赋值，开发者只需要将前端会传入的参数名称和后端类中的名称做好事先的确认即可。
然后就是通过反射调用execute方法，使用了Method.invoke方法。再次使用反射获取field的最新值，组成map返回，同时根据方法的返回值，去actionConfigMap中获取对应的view。
最后根据field的返回值map和view的名称组成最终展示的视图。

### 结尾
以上其实就是根据反射知识模仿的struts核心运行流程的小demo，整个web框架处理了非常多的其他的事情比如参数映射，安全，Json处理等，如果有兴趣，可以进一步做学习。


*demo地址*:[https://github.com/kailuncen/tdd-struts-refactor-demo](https://github.com/kailuncen/tdd-struts-refactor-demo)
