---
layout:     post
title:      "IDEA配置ApplicationContext.xml路径"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - IDEA
    - Spring
    - applicationContext
---

## 0 一篇一句

**任何没有计划的学习都只是作秀，任何没有走心的努力都是看起来很努力**

## 1 问题

IDEA加载Spring的上下文，读取applicationContext.xml文件的时候，出现找不到文件的情况。

```
java.io.FileNotFoundException:
class path resource [src/main/java/com/baobaotao/advice/beans.xml]
cannot be opened because it does not exist
```

## 2 解决

快捷键Command + ; 调出Project Structure    
『Project Settings --> Modules --> amaze-blender』    

1. 设置项目的resource文件夹。
右半部分，选择Source的Tab
![setResources](/img/in-post/post-idea-settings-with-spring-context/set-resources.png)

2. 设置项目的spring配置文件地址
![setSprintContextPath](/img/in-post/post-idea-settings-with-spring-context/set-spring-context-path.png)

3. 查看项目中，load的代码。很有可能被IDEA自动重构了！有点醉！
![badAutoRefacting](/img/in-post/post-idea-settings-with-spring-context/bad-auto-refacting.png)
![correctPath](/img/in-post/post-idea-settings-with-spring-context/correct-path.png)


## 3 Spring加载ApplicationContext的方式

四种方式：
1. XmlBeanFactory
2. ClassPathXmlApplicationContext
3. FileSystemXmlApplicationContext
4. XmlWebApplicationContext

## 4 参考

[【yifangyou】spring加载ApplicationContext.xml的四种方式](http://yifangyou.blog.51cto.com/900206/632078)    
[【极客学院】Spring ApplicationContext 容器](http://wiki.jikexueyuan.com/project/spring/ioc-container/spring-application-context-container.html)    
[【henu_zhangyang】加载spring上下文几种方式汇总](http://ihenu.iteye.com/blog/2268956)    
