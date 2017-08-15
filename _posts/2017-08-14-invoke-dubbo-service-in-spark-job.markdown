---
layout:     post
title:      "Spark任务中调用Dubbo服务"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Spark
    - Spring
    - Dubbo
---

## 0 一篇一句

**曾经有人问我，失去的东西回来了还要吗？**    
**我说，曾经丢了一粒扣子，等到找回那粒扣子时，我已经换了衣服。**

## 1 概述

#### 1.1 目标

**在Spark任务中，调用Dubbo服务** （不想听哔哔的可以直接跳到[『结论』](#summary)）    

#### 1.2 基础知识

- Dubbo服务依赖于Spring，要调用Dubbo的服务，需要加载 **Spring的上下文** 。    
(详见：[加载Spring上下文的多种方式](/2017/08/14/ways-to-load-spring-context/){:target="_blank_"})
- Spark的任务使用 [spark-submit](https://taoistwar.gitbooks.io/spark-operationand-maintenance-management/content/spark_install/spark_submmit.html) 指令来提交

#### 1.3 步骤

1. 使用ClassPathXmlApplicationContext，仿照blender的启动脚本，来写Spark的启动脚本。
    > 发现 spark-submit 没法把文件提交到 classpath 。

2. 改用FileSystemXmlApplicationContext。
    > 可以读到 applicationContext.xml 文件，但是， xsd 校验文件找不到。

3. 查找资料，发现http的地址已经找不到文件了。但是，dubbo.jar里面包含了dubbo.xsd
    > 解压 jar 之后，发现包含有 dubbo.xsd

4. 为什么已经包含，仍然报找不到的错？继续找资料，发现spring.schemas的资料。
    > 比较， blender 打出来的包里面的 spring.schemas ，与 Spark 项目打出来的包里面的 spring.schemas    
    > 发现 blender 打出来的里面有 dubbo.xsd 的路径配置，而 Spark 打出的里面没有 dubbo.xsd 的路径配置

5. 说明打包方式有问题。继续查找资料，使用maven-shade-plugin打包。
    > 使用 org.apache.maven.plugins.shade.resource.AppendingTransformer    
    > 往 META-INF/spring.handlers 和 META-INF/srping.schemas 里面追加内容

## 2 尝试详情

#### 2.1 使用ClassPathXmlApplicationContext

###### 2.1.1 加载Spring上下文的工具类
```java
package com.leappmusic.amaze.spark.utils;

import com.leapp.amaze.video.service.VideoService;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class DubboHelper {
    private static DubboHelper dubboHelper;
    private static VideoService videoService;

    private static ClassPathXmlApplicationContext context;

    private DubboHelper() {
        // 使用 ClassPathXmlApplicationContext 加载 Spring 上下文
        context = new ClassPathXmlApplicationContext("applicationContext.xml");
        videoService = context.getBean(VideoService.class);
        context.registerShutdownHook();
    }

    public static DubboHelper getInstance() {
        if (dubboHelper == null) {
            synchronized (DubboHelper.class) {
                if (dubboHelper == null) {
                    dubboHelper = new DubboHelper();
                }
            }
        }
        return dubboHelper;
    }

    public VideoService getVideoService() {
        return videoService;
    }
}
```

###### 2.1.2 Spark启动脚本 {#spark_submit_shell}
尝试过使用 --files 和 --jars 提交 applicationContext.xml 文件。都不能成功读取。
```
#!/bin/bash
if [ -f ./path_config.sh ]; then
    . ./path_config.sh
fi

hadoop fs -rmr $HACKER_NEWS_RANK_OUT*

unit=604800000

spark_master=local

spark-submit \
    --master $spark_master \
    --executor-memory 1g \
    --files applicationContext.xml \
    --driver-class-path mysql-connector-java-5.1.21.jar \
    --total-executor-cores 1 \
    --class com.leappmusic.amaze.spark.model.HackerNewsRank $spark_jar $TB_VIDEO_MONGO_DA $TB_VIDEO_MYSQL_DA $HACKER_NEWS_RANK_OUT 1.8 $unit
```

###### 2.1.3 报错 FileNotFoundException

意思就是，在 classpath 的路径下找不到 『applicationContext.xml』 这个文件。
```
17/08/10 18:50:31 INFO xml.XmlBeanDefinitionReader: Loading XML bean definitions from class path resource [applicationContext.xml]
Exception in thread "main" org.springframework.beans.factory.BeanDefinitionStoreException: IOException parsing XML document from class path resource [applicationContext.xml]; nested exception is java.io.FileNotFoundException: class path resource [applicationContext.xml] cannot be opened because it does not exist
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:344)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:304)
```

#### 2.2 改用FileSystemXmlApplicationContext {#FileSystemXmlApplicationContext}
查了很久spark-submit如何把文件提交到classpath，无果。    
所以，转向另外的思路：使用其他方式加载Spring上下文

```java
package com.leappmusic.amaze.spark.utils;

import com.leapp.amaze.video.service.VideoService;
import org.springframework.context.support.FileSystemXmlApplicationContext;

public class DubboHelper {
    private static DubboHelper dubboHelper;
    private static VideoService videoService;

    private static FileSystemXmlApplicationContext context;

    private DubboHelper() {
        // ！！！使用 FileSystemXmlApplicationContext 加载 Spring 上下文！！！
        context = new FileSystemXmlApplicationContext("applicationContext.xml");
        videoService = context.getBean(VideoService.class);
        context.registerShutdownHook();
    }

    public static DubboHelper getInstance() {
        if (dubboHelper == null) {
            synchronized (DubboHelper.class) {
                if (dubboHelper == null) {
                    dubboHelper = new DubboHelper();
                }
            }
        }
        return dubboHelper;
    }

    public VideoService getVideoService() {
        return videoService;
    }
}
```

###### 2.2.2 Spark启动脚本

未修改，与[『2.1.2节』](#spark_submit_shell)相同

###### 2.2.3 报错 Failed to read schema document {#failed_to_read_schema}
***找不到『dubbo.xsd』这个schema文件***    
> Failed to read schema document 'http://code.alibabatech.com/schema/dubbo/dubbo.xsd', because **1) could not find the document**; 2) the document could not be read; 3) the root element of the document is not <xsd:schema>.

```
17/08/11 11:22:55 INFO support.FileSystemXmlApplicationContext: Refreshing org.springframework.context.support.FileSystemXmlApplicationContext@61147f44: startup date [Fri Aug 11 11:22:55 CST 2017]; root of context hierarchy
17/08/11 11:22:55 INFO xml.XmlBeanDefinitionReader: Loading XML bean definitions from file [/home/leappspark/work/prod/servers/spark-apps-test/bin/applicationContext.xml]
7/08/11 11:23:02 WARN xml.XmlBeanDefinitionReader: Ignored XML validation warning
org.xml.sax.SAXParseException; lineNumber: 6; columnNumber: 72; schema_reference.4: Failed to read schema document 'http://code.alibabatech.com/schema/dubbo/dubbo.xsd', because 1) could not find the document; 2) the document could not be read; 3) the root element of the document is not <xsd:schema>.
	at org.apache.xerces.util.ErrorHandlerWrapper.createSAXParseException(Unknown Source)
	at org.apache.xerces.util.ErrorHandlerWrapper.warning(Unknown Source)
	at org.apache.xerces.impl.XMLErrorReporter.reportError(Unknown Source)
	at org.apache.xerces.impl.XMLErrorReporter.reportError(Unknown Source)
	at org.apache.xerces.impl.XMLErrorReporter.reportError(Unknown Source)
	at org.apache.xerces.impl.xs.traversers.XSDHandler.reportSchemaWarning(Unknown Source)
	at org.apache.xerces.impl.xs.traversers.XSDHandler.getSchemaDocument(Unknown Source)
	at org.apache.xerces.impl.xs.traversers.XSDHandler.parseSchema(Unknown Source)
	at org.apache.xerces.impl.xs.XMLSchemaLoader.loadSchema(Unknown Source)
	at org.apache.xerces.impl.xs.XMLSchemaValidator.findSchemaGrammar(Unknown Source)
	at org.apache.xerces.impl.xs.XMLSchemaValidator.handleStartElement(Unknown Source)
	at org.apache.xerces.impl.xs.XMLSchemaValidator.emptyElement(Unknown Source)
	at org.apache.xerces.impl.XMLNSDocumentScannerImpl.scanStartElement(Unknown Source)
	at org.apache.xerces.impl.XMLDocumentFragmentScannerImpl$FragmentContentDispatcher.dispatch(Unknown Source)
	at org.apache.xerces.impl.XMLDocumentFragmentScannerImpl.scanDocument(Unknown Source)
	at org.apache.xerces.parsers.XML11Configuration.parse(Unknown Source)
	at org.apache.xerces.parsers.XML11Configuration.parse(Unknown Source)
	at org.apache.xerces.parsers.XMLParser.parse(Unknown Source)
	at org.apache.xerces.parsers.DOMParser.parse(Unknown Source)
	at org.apache.xerces.jaxp.DocumentBuilderImpl.parse(Unknown Source)
	at org.springframework.beans.factory.xml.DefaultDocumentLoader.loadDocument(DefaultDocumentLoader.java:76)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReade
```

#### 2.3 查找dubbo.xsd
1. 地址 http://code.alibabatech.com/schema/dubbo/dubbo.xsd 已经找不到文件了
2. 为什么 url 访问不到，但是，在其他项目里面还可以正常使用dubbo？猜想，有本地文件。
3. 查找[资料](http://blog.csdn.net/wabiaozia/article/details/50491700)，发现，dubbo.jar里面包含有dubbo.xsd
```
# 解压jar
[leappspark@emr-header-1 tmp]$ jar xf spark-apps-0.1-SNAPSHOT-jar-with-dependencies.jar
# 查找"dubbo.xsd"
[leappspark@emr-header-1 tmp]$ find . -name "dubbo.xsd"
./META-INF/dubbo.xsd
```

#### 2.4 查看spring.schemas
在 2.3 节中，可以发现，Spark项目打出的jar包里面，已经包含了dubbo.xsd。但是，为什么仍然[报错 Failed to read schema document](#failed_to_read_schema)    
继续查找[资料](http://blog.csdn.net/houyefeng/article/details/44200995)。发现了一个映射文件『spring.schemas』。    

打开dubbo的jar包，可以在它的spring.schemas文件里看到有这样的配置：
```
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
```

> 注：这里就是 jar 里面已经包含了 dubbo.xsd ，但是，仍然读取不到的原因。

**比较 Spark项目的jar和blender项目的jar中，META-INF/spring.schemas文件。    
用来验证spring.schemas文件的作用：做schema的url与本地文件位置的映射**

###### 2.4.1 Spark项目中的spring.schemas文件
```
# 没有包含dubbo.xsd的配置
[leappspark@emr-header-1 tmp]$ vim META-INF/spring.schemas
http\://www.springframework.org/schema/context/spring-context-2.5.xsd=org/springframework/context/config/spring-context-2.5.xsd
http\://www.springframework.org/schema/context/spring-context-3.0.xsd=org/springframework/context/config/spring-context-3.0.xsd
http\://www.springframework.org/schema/context/spring-context-3.1.xsd=org/springframework/context/config/spring-context-3.1.xsd
http\://www.springframework.org/schema/context/spring-context-3.2.xsd=org/springframework/context/config/spring-context-3.2.xsd
http\://www.springframework.org/schema/context/spring-context-4.0.xsd=org/springframework/context/config/spring-context-4.0.xsd
http\://www.springframework.org/schema/context/spring-context-4.1.xsd=org/springframework/context/config/spring-context-4.1.xsd
http\://www.springframework.org/schema/context/spring-context-4.2.xsd=org/springframework/context/config/spring-context-4.2.xsd
http\://www.springframework.org/schema/context/spring-context-4.3.xsd=org/springframework/context/config/spring-context-4.3.xsd
http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context-4.3.xsd

# 其中，省略了一些...但是，没有dubbo.xsd的配置！

http\://www.springframework.org/schema/aop/spring-aop-2.0.xsd=org/springframework/aop/config/spring-aop-2.0.xsd
http\://www.springframework.org/schema/aop/spring-aop-2.5.xsd=org/springframework/aop/config/spring-aop-2.5.xsd
http\://www.springframework.org/schema/aop/spring-aop-3.0.xsd=org/springframework/aop/config/spring-aop-3.0.xsd
http\://www.springframework.org/schema/aop/spring-aop-3.1.xsd=org/springframework/aop/config/spring-aop-3.1.xsd
http\://www.springframework.org/schema/aop/spring-aop-3.2.xsd=org/springframework/aop/config/spring-aop-3.2.xsd
http\://www.springframework.org/schema/aop/spring-aop-4.0.xsd=org/springframework/aop/config/spring-aop-4.0.xsd
http\://www.springframework.org/schema/aop/spring-aop-4.1.xsd=org/springframework/aop/config/spring-aop-4.1.xsd
http\://www.springframework.org/schema/aop/spring-aop-4.2.xsd=org/springframework/aop/config/spring-aop-4.2.xsd
http\://www.springframework.org/schema/aop/spring-aop-4.3.xsd=org/springframework/aop/config/spring-aop-4.3.xsd
http\://www.springframework.org/schema/aop/spring-aop.xsd=org/springframework/aop/config/spring-aop-4.3.xsd

```

###### 2.4.2 blender项目中的spring.schemas文件
```
# 包含了dubbo.xsd的配置
[service@iZuf6j8qby6d0q404boi13Z tmp]$ vim META-INF/spring.schemas
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
```

#### 2.5 使用maven-shade-plugin

由2.4节可知，虽然，两个项目打出来的jar包里面都包含了dubbo.xsd文件。但是，一个可以找到另一个找不到。区别就在于『spring.schemas』文件。    
为什么，两个项目的spring.schemas文件不一样呢？原因，在于两个项目的打包方式不一样。    
blender用的maven-shade-plugin进行打包，而Spark项目使用的maven-assembly-plugin打包。

从Blog[『storm启动spring项目Failed to read schema document 'http://code.alibabatech.com/schema/dubbo/dubbo.xsd'处理』](http://blog.csdn.net/houyefeng/article/details/44200995)中得到启发。下面，是一段引用：
> 到此问题已经非常明朗了，问题就出在maven的打包插件maven-assembly-plugin上，经查找资料发现assembly将多个jar合并到一个jar中时，如果不同的jar中有同名的文件时会导致文件内容的丢失，丢失内容有一定的不确定性。
注：此问题不只出现在storm上部署应用时，在所有需要将依赖包的jar合并成一个jar发布的应用中都有可能出现。

**maven-assembly-plugin将多个jar合并到一个jar中时，如果不同的jar中有同名的文件时会导致文件内容的丢失，丢失内容有一定的不确定性。    
我猜想，策略可能有两种：『替换』或者『只保存第一个』**

**使用maven-shade-plugin打包，即可以解决问题。    
maven-shade-plugin遇到相同的文件，可以使用『追加』策略，所以可以解决问题。**

###### 2.5.1 java.lang.SecurityException

使用maven-shade-plugin打包的时候，出现java.lang.SecurityException。查找资料发现，跟jar包里面的签名校验文件有关，打包的时候把那些文件忽略就行。[详见](http://www.cnblogs.com/buptl/p/6774016.html)
```
Exception in thread "main" java.lang.SecurityException: Invalid signature file digest for Manifest main attributes
    at sun.security.util.SignatureFileVerifier.processImpl(SignatureFileVerifier.java:287)
    at sun.security.util.SignatureFileVerifier.process(SignatureFileVerifier.java:240)
    at java.util.jar.JarVerifier.processEntry(JarVerifier.java:317)
    at java.util.jar.JarVerifier.update(JarVerifier.java:228)
    at java.util.jar.JarFile.initializeVerifier(JarFile.java:348)
    at java.util.jar.JarFile.getInputStream(JarFile.java:415)
    at sun.misc.URLClassPath$JarLoader$2.getInputStream(URLClassPath.java:775)
    at sun.misc.Resource.cachedInputStream(Resource.java:77)
    at sun.misc.Resource.getByteBuffer(Resource.java:160)
    at java.net.URLClassLoader.defineClass(URLClassLoader.java:436)
    at java.net.URLClassLoader.access$100(URLClassLoader.java:71)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:361)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
    at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:482)
```

## 3 结论 {#summary}

1. DubboHelper中，使用FileSystemXmlApplicationContext加载Spring上下文。[参考](#FileSystemXmlApplicationContext)
2. spark-submit脚本中，使用 \-\-files 提交applicationContext.xml。[参考](#spark_submit_shell)
3. 使用maven-shade-plugin打包插件。 **注意 build 标签下面 maven-shade-plugin 的配置**
4. 使用mvn clean package生成jar包。（不能使用mvn clean -U assembly:assembly）

> 注：在所有需要将依赖包的 jar 合并成一个 jar 发布的应用中都应该使用maven-shade-plugin插件进行打包。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.leappmusic.recommend</groupId>
    <artifactId>spark-apps</artifactId>
    <version>0.1-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>2.0.0</version>
            <scope>provided</scope>
            <exclusions>
                <exclusion>
                    <artifactId>hadoop-client</artifactId>
                    <groupId>org.apache.hadoop</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.7.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-mllib_2.10</artifactId>
            <version>2.0.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.mongodb.spark</groupId>
            <artifactId>mongo-spark-connector_2.10</artifactId>
            <version>2.0.0</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.10</artifactId>
            <version>2.0.0</version>
            <scope>compile</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka_2.10</artifactId>
            <version>1.5.2</version>
        </dependency>
        <dependency>
            <groupId>com.leappmusic.amaze.serving</groupId>
            <artifactId>amaze-commons</artifactId>
            <version>0.4-SNAPSHOT</version>
            <exclusions>
                <exclusion>
                    <groupId>org.mongodb</groupId>
                    <artifactId>mongodb-driver</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.leappmusic.serving</groupId>
            <artifactId>serving-utils</artifactId>
            <version>0.1-SNAPSHOT</version>
            <exclusions>
                <exclusion>
                    <groupId>org.mongodb</groupId>
                    <artifactId>mongodb-driver</artifactId>
                </exclusion>
            </exclusions>
        </dependency>


        <!--aliyun-->
        <dependency>
            <groupId>com.aliyun.emr</groupId>
            <artifactId>emr-logservice_2.11</artifactId>
            <version>1.4.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.10</artifactId>
            <version>0.8.2.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.json4s/json4s-native_2.10 -->
        <dependency>
            <groupId>org.json4s</groupId>
            <artifactId>json4s-native_2.10</artifactId>
            <version>3.5.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-sql_2.10 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.10</artifactId>
            <version>2.0.0</version>
            <scope>compile</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpasyncclient</artifactId>
            <version>4.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore-nio</artifactId>
            <version>4.4.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore</artifactId>
            <version>4.4.1</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.3.7.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>com.leapp.amaze</groupId>
            <artifactId>amaze-video-api</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
        <!-- 引用 dubbo服务 -->
        <dependency>
            <groupId>com.github.sgroschupf</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.1</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.4.10</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/spring.handlers</resource>
                                </transformer>
                                <transformer  implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/spring.schemas</resource>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <profiles>
        <profile>
            <id>scala</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <scala.version>2.10.4</scala.version>
            </properties>
            <build>
                <plugins>
                    <plugin>
                        <groupId>net.alchim31.maven</groupId>
                        <artifactId>scala-maven-plugin</artifactId>
                        <version>3.2.1</version>
                        <executions>
                            <execution>
                                <goals>
                                    <goal>compile</goal>
                                    <goal>testCompile</goal>
                                </goals>
                            </execution>
                        </executions>
                        <configuration>
                            <scalaVersion>${scala.version}</scalaVersion>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>

```

## 4 遗留问题

Spark任务加入Dubbo的service调用之后，在业务逻辑执行完成之后，无法自动关闭。    
查资料，跟Dubbo的停机方式有关，关键词：**Shutdown Hook**    

**『 预知后事如何，请听下回分解 』**

## 5 感谢

感谢『水猴007』在CSDN上的博客：[storm启动spring项目Failed to read schema document 'http://code.alibabatech.com/schema/dubbo/dubbo.xsd'处理](http://blog.csdn.net/houyefeng/article/details/44200995)    
**这篇博客给了我很大的启发！帮我节省了不少时间！谢谢！**

## 6 参考

[【CSDN】Failed to read schema document 'http://code.alibabatech.com/schema/dubbo/dubbo.xsd'问题解决方法](http://blog.csdn.net/dczjzz/article/details/51697823)    
[【CSDN】storm启动spring项目Failed to read schema document 'http://code.alibabatech.com/schema/dubbo/dubbo.xsd'处理](http://blog.csdn.net/houyefeng/article/details/44200995)    
[【蓝色空间】解决fatjar的 “java.lang.SecurityException: Invalid signature file digest for Manifest main attributes” 问题](http://www.cnblogs.com/buptl/p/6774016.html)    
[【CSDN】java.lang.SecurityException: Invalid signature file digest for Manifest main attributes](http://blog.csdn.net/dhmj2ee/article/details/8893941)    
[【Stack Overflow】“Invalid signature file” when attempting to run a .jar](https://stackoverflow.com/questions/999489/invalid-signature-file-when-attempting-to-run-a-jar)    
[【Apache】Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/)    
