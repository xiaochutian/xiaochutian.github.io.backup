---
layout:     post
title:      "加载Spring上下文的多种方式"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Spring
    - applicationContext
---

## 0 一篇一句

**Make Yourself At Home**

## 1 不同的加载方式

- XmlBeanFactory
- ClassPathXmlApplicationContext
- FileSystemXmlApplicationContext
- XmlWebApplicationContext

#### 1.1 XmlBeanFactory
```java
// 引用资源用XmlBeanFactory（不能实现多个文件相互引用）
Resource resource = new ClassPathResource("applicationContext.xml");
BeanFactory context = new XmlBeanFactory(resource);
```

#### 1.2 ClassPathXmlApplicationContext
```java
// 引用应用上下文用ClassPathXmlApplicationContext
ApplicationContext context=new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
ApplicationContext context=new ClassPathXmlApplicationContext("conf/userConfig.xml");   
ApplicationContext context=new ClassPathXmlApplicationContext("file:G:/Test/src/appcontext.xml");
```

#### 1.3 FileSystemXmlApplicationContext
```java
// 用文件系统的路径引用应用上下文用FileSystemXmlApplicationContext
ApplicationContext context=new FileSystemXmlApplicationContext("src/applicationContext.xml");
ApplicationContext context=new FileSystemXmlApplicationContext("classpath:appcontext.xml");
ApplicationContext context=new FileSystemXmlApplicationContext("file:G:/Test/src/appcontext.xml");
ApplicationContext context=new FileSystemXmlApplicationContext("G:/Test/src/appcontext.xml");
```

#### 1.4 XmlWebApplicationContext
```java
// Web工程定制的加载方法 XmlWebApplicationContext
ServletContext servletContext = request.getSession().getServletContext();
ApplicationContext context = WebApplicationContextUtils.getWebApplicationContext(servletContext );
```

**web.xml配置**    
```xml
<servlet>  
    <servlet-name>app</servlet-name>  
    <servlet-class>  
        org.springframework.web.servlet.DispatcherServlet  
    </servlet-class>  
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>/WEB-INF/applicationContext*.xml,/WEB-INF/user_spring*.xml</param-value>  
    </context-param>  
    <load-on-startup>1</load-on-startup>    
</servlet>
```

## 2 其他

XmlWebApplicationContext加载Spring上下文的原理：    
[【简书】Spring 的根上下文和子上下文详解](http://www.jianshu.com/p/b3eab5acc7f4)

## 3 参考
[【Proli】Spring加载上下文几种方式（Spring配置XML）](http://www.cnblogs.com/proli/p/6760326.html)    
[【简书】Spring 的根上下文和子上下文详解](http://www.jianshu.com/p/b3eab5acc7f4)
