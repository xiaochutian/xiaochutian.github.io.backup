---
layout:     post
title:      "单例模式解决数据访问层"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Java
    - DAO
    - Singleton
---

## 0 一篇一句

**种一棵树最好的时间是十年前，其次是现在。**

## 1 概述

#### 1.1 DAO

数据访问对象模式（Data Access Object Pattern）或 DAO 模式用于把低级的数据访问 API 或操作从高级的业务服务中分离出来。以下是数据访问对象模式的参与者。
- 数据访问对象接口（Data Access Object Interface） - 该接口定义了在一个模型对象上要执行的标准操作。
- 数据访问对象实体类（Data Access Object concrete class） - 该类实现了上述的接口。该类负责从数据源获取数据，数据源可以是数据库，也可以是 xml，或者是其他的存储机制。
- 模型对象/数值对象（Model Object/Value Object） - 该对象是简单的 POJO，包含了 get/set 方法来存储通过使用 DAO 类检索到的数据。

#### 1.2 Singleton

单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。    
保证一个类仅有一个实例，并提供一个访问它的全局访问点。

Java如何编写单例，前人已经总结好了。    
推荐的方式有三种：
1. 双重检验锁模式（double checked locking pattern）『大家常用』
2. 静态内部类『《Effective Java》推荐』
3. 枚举『本人力荐』

详情请查看：[『如何正确地写出单例模式』](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)
> 注意：在使用双重检验锁模式的时候，变量使用volatile修饰，防止指重排序。

## 2 修改前的DAO层

#### 2.1 实体类 {#entity}
DAO层负责对象的存取。有EntityA，和EntityB这两个对象。    
实体类中的具体属性不需要关注，因为，是对整个对象进行序列化和反序列化。

```java
public class EntityA implements Serializable {
    private static final long serialVersionUID = 1123581321L;
    // properties
}

public class EntityB implements Serializable {
    private static final long serialVersionUID = 2358132134L;
    // properties
}
```
#### 2.2 DAO类
在修改前，EntityA和，EntityB分别有不同的DAO对象负责存取。    
不同的实体类，有不同的存储地址。但是，存取方式都是统一的，使用对象序列化和反序列化。

```java
public class EntityADAO {
    private static final Logger logger = LoggerFactory.getLogger(EntityADAO.class);
    private volatile static EntityADAO instance;

    private JedisHelper jedisHelper;

    public static EntityADAO getInstance() {
        if (instance == null) {
            synchronized (EntityADAO.class) {
                if (instance == null) {
                    instance = new EntityADAO();
                }
            }
        }
        return instance;
    }

    private EntityADAO() {
        ConfigHelper config = new ConfigHelper("dao.properties");
        String redisAddress = config.getString("EntityA.dao.address");

        jedisHelper = new JedisHelper(redisAddress);
    }

    public EntityA getEntityA(String key) {
        try {
            byte[] bytes = jedisHelper.get(key.getBytes());
            if (bytes == null) {
                return null;
            }
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
            ObjectInputStream inputStream = new ObjectInputStream(byteArrayInputStream);
            return (EntityA) inputStream.readObject();
        } catch (JedisHelper.JedisHelperException e) {
            logger.error("jedis get error", e);
        } catch (ClassNotFoundException | IOException e) {
            logger.error("deserialize error", e);
        }
        return null;
    }

    public void setEntityA(String key, EntityA entityA) {
        try {
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
            objectOutputStream.writeObject(entityA);
            byte[] bytes = byteArrayOutputStream.toByteArray();
            jedisHelper.set(key.getBytes(), bytes);
        } catch (JedisHelper.JedisHelperException e) {
            logger.error("jedis set error", e);
        } catch (IOException e) {
            logger.error("serialize error", e);
        }
    }
}

public class EntityBDAO {
    private static final Logger logger = LoggerFactory.getLogger(EntityBDAO.class);
    private volatile static EntityBDAO instance;

    private JedisHelper jedisHelper;

    public static EntityBDAO getInstance() {
        if (instance == null) {
            synchronized (EntityBDAO.class) {
                if (instance == null) {
                    instance = new EntityBDAO();
                }
            }
        }
        return instance;
    }

    private EntityBDAO() {
        ConfigHelper config = new ConfigHelper("dao.properties");
        String redisAddress = config.getString("EntityB.dao.address");

        jedisHelper = new JedisHelper(redisAddress);
    }

    public EntityB getEntityB(String key) {
        try {
            byte[] bytes = jedisHelper.get(key.getBytes());
            if (bytes == null) {
                return null;
            }
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
            ObjectInputStream inputStream = new ObjectInputStream(byteArrayInputStream);
            return (EntityB) inputStream.readObject();
        } catch (JedisHelper.JedisHelperException e) {
            logger.error("jedis get error", e);
        } catch (ClassNotFoundException | IOException e) {
            logger.error("deserialize error", e);
        }
        return null;
    }

    public void setEntityB(String key, EntityB entityB) {
        try {
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
            objectOutputStream.writeObject(entityB);
            byte[] bytes = byteArrayOutputStream.toByteArray();
            jedisHelper.set(key.getBytes(), bytes);
        } catch (JedisHelper.JedisHelperException e) {
            logger.error("jedis set error", e);
        } catch (IOException e) {
            logger.error("serialize error", e);
        }
    }
}
```
#### 2.3 存在的问题

在get和set方法里面，代码几乎是一样的。**IDEA编译器会提示，有重复代码。**    
在最开始的时候，我也就照着原有的代码添加了一两个实体类和DAO类。    
但是，添加到第三个的时候，总觉得哪里不对。于是，下定决心重构代码。

**一定不要用copy-past方式来写代码，见到别人写的这个，要想办法重构。**

## 3 修改

#### 3.1 修改思路
1. 把『实体类的类名』和『实体类的存储地址』关联起来，写入配置文件里面。（不同部分）
2. 使用 **『反射』** 和 **『泛型』** 来处理实体类的存取过程。（相同的部分）

#### 3.2 修改后

###### 3.2.1 实体类
不发生变化，与 [【2.1】](#entity)节中的相同。

###### 3.2.2 DAO类
包括『BaseMongo』和『MongoManager』

```java
public class MongoManager {
    public static <T> BaseMongo<T> getInstance(Class<T> classOf) {
        return BaseMongo.getInstance(classOf);
    }
}

public class BaseMongo<T> {
    private static final Logger logger = LoggerFactory.getLogger(BaseMongo.class);
    private volatile static Map<String, BaseMongo> instanceMap = new HashMap<>();
    private MongoCollection<Document> collection;

    private BaseMongo(String className) {
        // 加载配置文件
        ConfigHelper config = new ConfigHelper("application.dao.properties");
        // 从配置文件中读取mongo的地址和db名（这部分，EntityA和EntityB相同）
        String mongoAddress = config.getString("application.mongo.dao.address");
        String db = config.getString("application.mongo.dao.db");
        // 从配置文件中读取『实体类』的Collection名（这部分，EntityA和EntityB不同）
        String mongoCollection = config.getString(className + "Mongo.collection");
        MongoClient mongoClient = new MongoClient(new MongoClientURI(mongoAddress));
        collection = mongoClient.getDatabase(db).getCollection(mongoCollection);
    }

    public static BaseMongo getInstance(Class c) {
        String className = c.getSimpleName();
        if (!instanceMap.containsKey(className)) {
            synchronized (BaseMongo.class) {
                if (!instanceMap.containsKey(className)) {
                    instanceMap.put(className, new BaseMongo(className));
                }
            }
        }
        return instanceMap.get(className);
    }

    public T get(String id) {
        try {
            Document document = collection.find(new BasicDBObject("_id", id)).first();
            byte[] bytes = ((Binary) document.get("value")).getData();
            try (ObjectInputStream objectInputStream = new ObjectInputStream(new ByteArrayInputStream(bytes))) {
                return (T) objectInputStream.readObject();
            }
        } catch (Exception e) {
            logger.error(e.getMessage());
        }
        return null;
    }

    public boolean set(String id, Object t) {
        try (ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
             ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream)) {
            objectOutputStream.writeObject(t);
            byte[] bytes = byteArrayOutputStream.toByteArray();
            return set(id, bytes);
        } catch (Exception e) {
            logger.error(e.getMessage());
        }
        return false;
    }

    public boolean set(String id, byte[] bytes) {
        if (bytes == null || id == null || id.equals("")) {
            return false;
        }
        try {
            Document document = new Document();
            document.put("_id", id);
            document.put("value", bytes);
            collection.replaceOne(new BasicDBObject("_id", id), document, new UpdateOptions().upsert(true));
            return true;
        } catch (Exception e) {
            logger.error(e.getMessage());
        }
        return false;
    }
}
```    

###### 3.2.3 配置文件

application.dao.properties的文件内容如下：    
```java
application.mongo.dao.address=mongodb://username:password@host:port
application.mongo.dao.db=DBName
EntityAMongo.collection=EntityA
EntityBMongo.collection=EntityB
```

> 注：(实体类的地址的key的命名规则，实体类的simpleName + "Mongo.collection")


## 4 本地Cache + Redis + Mongo 三层DAO结构

啊勒，明天写

## 5 参考
[【Wikipedia】Data access object](https://en.wikipedia.org/wiki/Data_access_object)    
[【菜鸟教程】数据访问对象模式](http://www.runoob.com/design-pattern/data-access-object-pattern.html)    
[【Jark's Blog】如何正确地写出单例模式](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)    
[【极客学院】单例模式](http://wiki.jikexueyuan.com/project/java-design-pattern/singleton-pattern.html)    
[【菜鸟教程】单例模式](http://www.runoob.com/design-pattern/singleton-pattern.html)    
