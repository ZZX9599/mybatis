# Mybatis重难点

# 0:数据准备

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for t_clazz
-- ----------------------------
DROP TABLE IF EXISTS `t_clazz`;
CREATE TABLE `t_clazz`  (
  `cid` int NOT NULL,
  `cname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  PRIMARY KEY (`cid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of t_clazz
-- ----------------------------
INSERT INTO `t_clazz` VALUES (1000, '高三一班');
INSERT INTO `t_clazz` VALUES (1001, '高三二班');
INSERT INTO `t_clazz` VALUES (2000, '高三三班');

SET FOREIGN_KEY_CHECKS = 1;
```



学生表：

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for t_student
-- ----------------------------
DROP TABLE IF EXISTS `t_student`;
CREATE TABLE `t_student`  (
  `sid` int NOT NULL,
  `sname` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL,
  `cid` int NULL DEFAULT NULL,
  PRIMARY KEY (`sid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of t_student
-- ----------------------------
INSERT INTO `t_student` VALUES (1, '张三', 1000);
INSERT INTO `t_student` VALUES (2, '李四', 1000);
INSERT INTO `t_student` VALUES (3, '王五', 1000);
INSERT INTO `t_student` VALUES (4, '赵六', 1001);
INSERT INTO `t_student` VALUES (5, '钱七', 1001);

SET FOREIGN_KEY_CHECKS = 1;
```

- StudentMapper接口

```java
/**
 * 根据学生ID查询学生信息和对应班级信息
 * @param sid
 * @return
 */
Student selectBySid01(Integer sid);
```

- 实体类

```java
@Data
public class Student {
    private Integer sid;
    private String sname;
    private Integer cid;
    private Clazz clazz;
}
```

```java
@Data
public class Clazz {
    private Integer cid;
    private String cname;
}
```

- 工具类

```java
public class SqlSessionUtil {
    private static SqlSessionFactory sqlSessionFactory = null;
    private SqlSessionUtil(){

    }
    static {
        try {
            sqlSessionFactory = new SqlSessionFactoryBuilder()
                    .build(Resources.getResourceAsStream("mybatis-config.xml"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static SqlSession getSqlSession(){
        return sqlSessionFactory.openSession(true);
    }
}
```

# 1:多对一映射

多对一：多是主体，则在多的一方，也就是Student里面加上一的属性，也就是`Clazz clazz`

多个学生属于一个班级，一对多，多的表加外键

需求：根据学生的ID查询学生的信息和对应的班级信息



## 1.1:实现方式一

实现方式：级联属性映射

```xml
<resultMap id="studentResultMap01" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <!-- 把cid查询的结果赋值给Student对象的clazz的cid属性-->
    <result property="clazz.cid" column="cid"/>
    <!-- 把cname查询的结果赋值给Student对象的clazz的cname属性-->
    <result property="clazz.cname" column="cname"/>
</resultMap>

<!--方式一：级联属性映射-->
<select id="selectBySid01" resultMap="studentResultMap01">
    select
        s.sid,s.sname,c.cid,c.cname
    from
        t_student s join t_clazz c on s.cid=c.cid
    where
        sid=#{sid}
</select>
```

- 测试类

```java
@Test
public void test01(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student = mapper.selectBySid01(1);
    System.out.println(student);
}
```

```sql
select s.sid,s.sname,c.cid,c.cname from t_student s join t_clazz c on s.cid=c.cid where sid=?
```

其实就是把查询到的 sid,sname,cid,cname分别赋值给Student对象的 sid,sname,clazz.cid,clazz.cname属性



## 1.2:实现方式二

使用 association 来定义关联的关系

其他位置都不需要修改，只需要修改resultMap中的配置：association即可

```xml
<resultMap id="studentResultMap02" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <!-- association:关联，学生对象关联Clazz对象的信息 properties:属性名 javaType:类型-->
    <association property="clazz" javaType="Clazz">
        <id property="cid" column="cid"/>
        <id property="cname" column="cname"/>
    </association>
</resultMap>

<!-- 方式二：association-->
<select id="selectBySid02" resultMap="studentResultMap02">
    select
        s.sid,s.sname,c.cid,c.cname
    from
        t_student s join t_clazz c on s.cid=c.cid
    where
        sid=#{sid}
</select>
```

其实也是把查询到的 sid,sname,cid,cname分别赋值给Student对象的 sid,sname,clazz.cid,clazz.cname属性



## 1.3:实现方式三

使用 association来实现分步查询

其他位置不需要修改，只需要修改以及添加以下三处：

- 第一处：association中select位置填写sqlId，sqlId=namespace+id，select就是下一条执行的SQL语句

  其中column属性作为这条子sql语句的条件

```xml
<resultMap id="studentResultMap03" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <association property="clazz"
                 select="com.zzx.mybatis.mapper.ClazzMapper.selectByCid"
                 column="cid"/>
</resultMap>

<!-- 方式三-->
<select id="selectBySid03" resultMap="studentResultMap03">
    select sid,sname,cid from t_student where sid=#{sid}
</select>
```



- 第二处：在ClazzMapper接口中添加方法

```java
public interface ClazzMapper {

    /**
     * 根据cid获取Clazz信息
     * @param cid
     * @return
     */
    Clazz selectByCid(Integer cid);
}
```

- 第三处：在ClazzMapper.xml文件中进行配置

```xml
<resultMap id="clazzResultMap" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
</resultMap>

<select id="selectByCid" resultMap="clazzResultMap">
    select cid,cname from t_clazz where cid=#{cid}
</select>
```



- 测试：

```java
@Test
public void test03(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student = mapper.selectBySid03(3);
    System.out.println(student);
}
```

```sql
Preparing: select sid,sname,cid from t_student where sid=?
Parameters: 3(Integer)

Preparing: select cid,cname from t_clazz where cid=?
Parameters: 1000(Integer)
```

执行了两条SQL语句

实际上就是`StudentMapper`那条SQL语句执行完成之后，执行`association`里面的`select`的方法

分步优点：

- 第一个优点：代码复用性增强
- 第二个优点：支持延迟加载。【暂时访问不到的数据可以先不查询。提高程序的执行效率】



## 1.4:延迟加载

要想支持延迟加载，非常简单，只需要在association标签中添加fetchType="lazy"即可

```xml
<!-- 延迟加载 -->
<resultMap id="studentResultMap03" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <association property="clazz"
                 select="com.zzx.mybatis.mapper.ClazzMapper.selectByCid"
                 fetchType="lazy"
                 column="cid"/>
</resultMap>

<select id="selectBySid03" resultMap="studentResultMap03">
    select sid,sname,cid from t_student where sid=#{sid}
</select>
```

现在只查询学生的姓名，不涉及到学生的班级信息

```java
@Test
public void test04(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student = mapper.selectBySid03(3);
    System.out.println(student.getSname());
}
```

结果发现：并不会执行`select`的语句

```sql
Preparing: select sid,sname,cid from t_student where sid=?
==> Parameters: 3(Integer)
```



如果后续需要使用到学生所在班级的名称，这个时候才会执行关联的sql语句，修改测试程序：

```java
@Test
public void test04(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student = mapper.selectBySid03(3);
    System.out.println(student.getSname());
    System.out.println(student.getClazz());
}
```

```sql
Preparing: select sid,sname,cid from t_student where sid=?
==> Parameters: 3(Integer)

Preparing: select cid,cname from t_clazz where cid=?
==> Parameters: 1000(Integer)
```

通过以上的执行结果可以看到，只有当使用到班级名称之后，才会执行关联的sql语句，这就是延迟加载



上面是指定某个语句开启延迟加载

在mybatis中如何开启全局的延迟加载呢？需要setting配置，如下：

![image-20221002112230086](https://zzx-note.oss-cn-beijing.aliyuncs.com/mybatis/image-20221002112230086.png)

```xml
<settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/>
    <setting name="lazyLoadingEnabled" value="true"/>
</settings>
```

配置一个这样的配置，就可以去掉 `association`里面的 `feachType`了

开启全局延迟加载之后，所有的sql都会支持延迟加载，如果某个sql你不希望它支持延迟加载怎么办呢？

可以将某个具体的配置的fetchType设置为eager

这样的话，针对某个特定的sql，你就关闭了延迟加载机制。

后期我们要不要开启延迟加载机制，主要看实际的业务需求是怎样的。



# 2:一对多映射

一对多：一是主体，则在一的一方，也就是Clazz里面加上一的属性，也就是`List<Student>`

一对多的实现，通常是在一的一方中有List集合属性

在Clazz类中添加List<Student> 属性

```java
@Data
public class Clazz {
    private Integer cid;
    private String cname;
    private List<Student> studentList;
}
```

一对多的实现通常包括两种实现方式：

- 第一种方式：collection
- 第二种方式：分步查询



## 2.1:实现方式一

collection实现方式

- 接口

```java
/**
 * 根据班级编号查询班级信息。同时班级中所有的学生信息也要查询。
 * @param cid
 * @return
 */
Clazz selectClazzAndStudentListByCid(Integer cid);
```

- 配置

```xml
<resultMap id="clazzResultMap" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
    <!-- ofType:集合里面存储的元素的类型 -->
    <collection property="studentList" ofType="Student">
        <id property="sid" column="sid"/>
        <result property="sname" column="sname"/>
    </collection>
</resultMap>

<select id="selectClazzAndStudentListByCid" resultMap="clazzResultMap">
    select c.cid,c.cname,s.sid,s.sname from t_clazz c join t_student s on c.cid = s.cid where c.cid = #{cid}
</select>    
```

本质就是把每一个查到的  sid，sname 赋值给Clazz对象的Student的属性



## 2.2:第二种方式

分步查询

先根据教室ID查询到教室信息，再根据教室ID查询学生信息，再把结果赋值给StudentList

- StudentMapper接口

```java
/**
 * 根据学生表的教室编号查询学生
 * @param id
 * @return
 */
Student selectByCid(Integer id);
```

- ClazzMapper接口

```java
/**
 * 根据cid获取Clazz信息
 * @param cid
 * @return
 */
Clazz selectByCid(Integer cid);
```

- ClazzMapper.xml

```xml
<resultMap id="clazzResultMap" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
    <!-- ofType:集合里面存储的元素的类型 -->
    <collection property="studentList"
                select="com.zzx.mybatis.mapper.StudentMapper.selectByCid"
                column="cid">
    </collection>
</resultMap>
<select id="selectByCid" resultMap="clazzResultMap">
    select cid,cname from t_clazz where cid=#{cid}
</select>
```

- StudentMapper.xml

```xml
<resultMap id="studentResultMap" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
</resultMap>
<select id="selectByCid" resultMap="studentResultMap">
    select sid,sname from t_student where cid = #{cid}
</select>
```

- 测试

```java
@Test
public void test05(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    ClazzMapper mapper = sqlSession.getMapper(ClazzMapper.class);
    Clazz clazz = mapper.selectByCid(1001);
    System.out.println(clazz);
}
```

```sql
Preparing: select cid,cname from t_clazz where cid=?
==> Parameters: 1001(Integer)

Preparing: select sid,sname from t_student where cid = ?
====> Parameters: 1001(Integer)
```

## 2.3:延迟加载

一对多延迟加载机制和多对一是一样的。同样是通过两种方式：

- 第一种：fetchType="lazy"

- 第二种：修改全局的配置setting，**lazyLoadingEnabled=true，**如果开启全局延迟加载的情况下

  想让某个sql不使用延迟加载：fetchType="eager"



# 3:Mybatis缓存

## 3.1:介绍

缓存：cache

缓存的作用：通过减少IO的方式，来提高程序的执行效率。

mybatis的缓存：将select语句的查询结果放到缓存（内存）当中，下一次还是这条select语句的话，直接从缓存中取，不再查数据库。一方面是减少了IO。另一方面不再执行繁琐的查找算法。效率大大提升。

mybatis缓存包括：

- 一级缓存：将查询到的数据存储到SqlSession中【会话级别】
- 二级缓存：将查询到的数据存储到SqlSessionFactory中【数据库级别】
- 或者集成其它第三方的缓存：比如EhCache【Java语言开发的】、Memcache【C语言开发的】等。

**缓存只针对于DQL语句，也就是说缓存机制只对应select语句**



## 3.2:一级缓存

一级缓存默认是开启的。不需要做任何配置。

原理：只要使用同一个SqlSession对象执行同一条SQL语句，就会走缓存。

```java
/**
 * 根据学生id查询学生的信息
 * @param sid
 * @return
 */
Student selectBySid(Integer sid);
```

```java
<select id="selectBySid" resultType="com.zzx.mybatis.pojo.Student">
    select * from t_student where sid=#{sid}
</select>
```

```java
@Test
public void test06(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student1 = mapper.selectBySid(1);
    Student student2 = mapper.selectBySid(1);
    Student student3 = mapper.selectBySid(1);
}
```

```sql
==>  Preparing: select * from t_student where sid=?
==> Parameters: 1(Integer)
```

发现只执行了一个SQL语句



## 3.3:一级缓存失效

什么情况下不走缓存？

- 第一种：不同的SqlSession对象【因为一级缓存是属于SqlSession会话级别的】
- 第二种：查询条件变化了【相当于执行了insert update delete 之一，数据可能更新，缓存也需要更新】



一级缓存失效情况包括两种：

- 第一种：第一次查询和第二次查询之间，手动清空了一级缓存

`sqlSession.clearCache()`

- 第二种：第一次查询和第二次查询之间，执行了增删改操作【这个增删改和哪张表没有关系，只要有insert delete update操作，一级缓存就失效】



## 3.3:二级缓存

二级缓存的范围是SqlSessionFactory，数据库级别的

使用二级缓存需要具备以下几个条件：

1. <setting name="cacheEnabled" value="true"> 全局性地开启或关闭所有映射器配置文件中已配置的任何缓存。默认就是true，无需设置。

2. 在需要使用二级缓存的SqlMapper.xml文件中添加配置：<cache />

3. 使用二级缓存的实体类对象必须是可序列化的，也就是必须实现java.io.Serializable接口

4. SqlSession对象关闭或提交之后，一级缓存中的数据才会被写入到二级缓存当中。此时二级缓存才可

   用，当会话对象提交或者关闭了，就会把缓存的数据同步到二级缓存里面



注意：mybatis优先使用一级缓存，因为会话级别的离该次操作更近，读取一级缓存不存在，并且符合上面

的条件之后，就会查询二级缓存



测试二级缓存：

条件一：默认打开，可以不管

条件二：在StudentMapper.xml里面添加 `<cache/>`

条件三：实现序列化接口

```java
@Data
public class Student implements Serializable {
    private static final long serialVersionUID = -797048528046769481L;
    private Integer sid;
    private String sname;
    private Integer cid;
    private Clazz clazz;
}
```

条件四：SqlSession对象关闭或提交之后，同步缓存数据到二级缓存里面

```java
@Test
public void testSelectBySId() throws Exception{
    // 这里只有一个SqlSessionFactory对象。二级缓存对应的就是SqlSessionFactory。
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder()
            .build(Resources.getResourceAsStream("mybatis-config.xml"));
    SqlSession sqlSession1 = sqlSessionFactory.openSession();
    SqlSession sqlSession2 = sqlSessionFactory.openSession();
    StudentMapper mapper1 = sqlSession1.getMapper(StudentMapper.class);
    StudentMapper mapper2 = sqlSession2.getMapper(StudentMapper.class);
    // 这行代码执行结束之后，实际上数据是缓存到一级缓存当中了。（sqlSession1是一级缓存。）
    Student student1 = mapper1.selectBySid(1);
    System.out.println(student1);
    // 如果这里不关闭SqlSession1对象的话，二级缓存中还是没有数据的。
    // 如果执行了这两行代码其中一个，sqlSession1的一级缓存中的数据会放到二级缓存当中。
    sqlSession1.close();
    // sqlSession1.commit();
    // 这行代码执行结束之后，实际上数据会缓存到一级缓存当中。（sqlSession2是一级缓存。）
    Student student2 = mapper2.selectBySid(1);
    System.out.println(student2);
    // 程序执行到这里的时候，会将sqlSession1这个一级缓存中的数据写入到二级缓存当中。
    // sqlSession1.close();
    // 程序执行到这里的时候，会将sqlSession2这个一级缓存中的数据写入到二级缓存当中。
    sqlSession2.close();
}
```



## 3.4:二级缓存失效

**二级缓存的失效：只要两次查询之间出现了增删改操作。二级缓存就会失效。【一级缓存也会失效】**



## 3.5:二级缓存的相关配置

- eviction：指定从缓存中移除某个对象的淘汰算法。默认采用LRU策略。

1. - LRU：Least Recently Used。最近最少使用。优先淘汰在间隔时间内使用频率最低的对象
   - LFU，最不常用，可能最近用得很多，但是整体用的最少

2. - FIFO：First In First Out。一种先进先出的数据缓存器。先进入二级缓存的对象最先被淘汰。

3. - SOFT：软引用。淘汰软引用指向的对象。具体算法和JVM的垃圾回收算法有关。

4. - WEAK：弱引用。淘汰弱引用指向的对象。具体算法和JVM的垃圾回收算法有关。

- flushInterval：

1. - 二级缓存的刷新时间间隔。单位毫秒。如果没有设置。就代表不刷新缓存，只要内存足够大，一直会向二级缓存中缓存数据。除非执行了增删改。

- readOnly：

1. - true：多条相同的sql语句执行之后返回的对象是共享的同一个。性能好。但是多线程并发可能会存在安全问题。

2. - false：多条相同的sql语句执行之后返回的对象是副本，调用了clone方法。性能一般。但安全。

- size：

1. - 设置二级缓存中最多可存储的java对象数量。默认值1024。



## 3.6:集成EhCache

集成EhCache是为了代替mybatis自带的二级缓存。一级缓存是无法替代的。

mybatis对外提供了接口，也可以集成第三方的缓存组件。比如EhCache、Memcache等。都可以。

EhCache是Java写的。Memcache是C语言写的。所以mybatis集成EhCache较为常见，按照以下步骤操作

就可以完成集成：

第一步：引入mybatis整合ehcache的依赖

```xml
<!--mybatis集成ehcache的组件-->
<dependency>
  <groupId>org.mybatis.caches</groupId>
  <artifactId>mybatis-ehcache</artifactId>
  <version>1.2.2</version>
</dependency>
```

第二步：在类的根路径下新建echcache.xml文件，并提供以下配置信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false">
    <!--磁盘存储:将缓存中暂时不使用的对象,转移到硬盘,类似于Windows系统的虚拟内存-->
    <diskStore path="e:/ehcache"/>
  
    <!--defaultCache：默认的管理策略-->
    <!--eternal：设定缓存的elements是否永远不过期。如果为true，则缓存的数据始终有效，如果为false那么还要根据timeToIdleSeconds，timeToLiveSeconds判断-->
    <!--maxElementsInMemory：在内存中缓存的element的最大数目-->
    <!--overflowToDisk：如果内存中数据超过内存限制，是否要缓存到磁盘上-->
    <!--diskPersistent：是否在磁盘上持久化。指重启jvm后，数据是否有效。默认为false-->
    <!--timeToIdleSeconds：对象空闲时间(单位：秒)，指对象在多长时间没有被访问就会失效。只对eternal为false的有效。默认值0，表示一直可以访问-->
    <!--timeToLiveSeconds：对象存活时间(单位：秒)，指对象从创建到失效所需要的时间。只对eternal为false的有效。默认值0，表示一直可以访问-->
    <!--memoryStoreEvictionPolicy：缓存的3 种清空策略-->
    <!--FIFO：first in first out (先进先出)-->
    <!--LFU：Less Frequently Used (最少使用).意思是一直以来最少被使用的。缓存的元素有一个hit 属性，hit 值最小的将会被清出缓存-->
    <!--LRU：Least Recently Used(最近最少使用). (ehcache 默认值).缓存的元素有一个时间戳，当缓存容量满了，而又需要腾出地方来缓存新的元素的时候，那么现有缓存元素中时间戳离当前时间最远的元素将被清出缓存-->
    <defaultCache eternal="false" maxElementsInMemory="1000" overflowToDisk="false" diskPersistent="false"
                  timeToIdleSeconds="0" timeToLiveSeconds="600" memoryStoreEvictionPolicy="LRU"/>

</ehcache>
```

第三步：修改SqlMapper.xml文件中的<cache/>标签，添加type属性

```xml
<cache type="org.mybatis.caches.ehcache.EhcacheCache"/>
```

第四步：编写测试程序使用



# 4:逆向工程

## 4.1:介绍

所谓的逆向工程是：根据数据库表逆向生成Java的pojo类，SqlMapper.xml文件，以及Mapper接口类等。

要完成这个工作，需要借助别人写好的逆向工程插件。

思考：使用这个插件的话，需要给这个插件配置哪些信息？

- pojo类名、包名以及生成位置。
- SqlMapper.xml文件名以及生成位置。
- Mapper接口名以及生成位置。
- 连接数据库的信息。
- 指定哪些表参与逆向工程。



## 4.2:使用步骤

- pom中添加逆向工程插件

```xml
<!--定制构建过程-->
<build>
  <!--可配置多个插件-->
  <plugins>
    <!--其中的一个插件：mybatis逆向工程插件-->
    <plugin>
      <!--插件的GAV坐标-->
      <groupId>org.mybatis.generator</groupId>
      <artifactId>mybatis-generator-maven-plugin</artifactId>
      <version>1.4.1</version>
      <!--允许覆盖-->
      <configuration>
        <overwrite>true</overwrite>
      </configuration>
      <!--插件的依赖-->
      <dependencies>
        <!--mysql驱动依赖-->
        <dependency>
          <groupId>mysql</groupId>
          <artifactId>mysql-connector-java</artifactId>
          <version>8.0.30</version>
        </dependency>
      </dependencies>
    </plugin>
  </plugins>
</build>
```

- 配置generatorConfig.xml

该文件名必须叫做：generatorConfig.xml

该文件必须放在类的根路径下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!--
        targetRuntime有两个值：
            MyBatis3Simple：生成的是基础版，只有基本的增删改查。
            MyBatis3：生成的是增强版，除了基本的增删改查之外还有复杂的增删改查。
    -->
    <context id="DB2Tables" targetRuntime="MyBatis3">
        <!--防止生成重复代码-->
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>

        <commentGenerator>
            <!--是否去掉生成日期-->
            <property name="suppressDate" value="true"/>
            <!--是否去除注释-->
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <!--连接数据库信息-->
        <jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/mybatis"
                        userId="root"
                        password="JXLZZX79">
        </jdbcConnection>

        <!-- 生成pojo包名和位置 -->
        <javaModelGenerator targetPackage="com.zzx.mybatis.pojo" targetProject="src/main/java">
            <!--是否开启子包-->
            <property name="enableSubPackages" value="true"/>
            <!--是否去除字段名的前后空白-->
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- 生成SQL映射文件的包名和位置 -->
        <sqlMapGenerator targetPackage="com.zzx.mybatis.mapper" targetProject="src/main/resources">
            <!--是否开启子包-->
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <!-- 生成Mapper接口的包名和位置 -->
        <javaClientGenerator
                type="xmlMapper"
                targetPackage="com.zzx.mybatis.mapper"
                targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- 表名和对应的实体类名-->
        <table tableName="t_car" domainObjectName="Car"/>

    </context>
</generatorConfiguration>
```

- 测试

```java
@Test
public void test01(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    Car car = mapper.selectByPrimaryKey(210L);
    System.out.println(car);
}
```

- 日志

```sql
==> Preparing: select id, car_num, brand, guide_price, produce_time, car_type from t_car where id = ?
==> Parameters: 210(Long)
```



## 4.3:XXExample

MyBatis3Simple：生成的是基础版，只有基本的增删改查。

MyBatis3：生成的是增强版，除了基本的增删改查之外还有复杂的增删改查。

我们发现，使用增强版生成的实体类多了一个XXExmaple，我这里是CarMapper

![image-20221002175719063](https://zzx-note.oss-cn-beijing.aliyuncs.com/mybatis/image-20221002175719063.png)

发现很多的方法都可以传递 带有 Example 的参数

其实带有这个Example的实体类就是给我们来封装参数信息的





如果想封装 `and`的信息，就要使用对象的 `createCriteria`方法

![image-20221002180002840](https://zzx-note.oss-cn-beijing.aliyuncs.com/mybatis/image-20221002180002840.png)

想要封装 or 的信息，就要使用对象的 `or` 方法



```java
@Test
public void test02(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    CarExample carExample=new CarExample();
    carExample.createCriteria()
            .andBrandEqualTo("兰博基尼")
            .andCarNumEqualTo("2001");
    carExample.or().andProduceTimeIsNull();
    mapper.selectByExample(carExample);
}
```

```sql
==>  Preparing: select id, car_num, brand, guide_price, produce_time, car_type from t_car WHERE ( brand = ? and car_num = ? ) or( produce_time is null )

==> Parameters: 兰博基尼(String), 2001(String)
```

# 5:动态SQL

## 5.1:准备工作

准备表结构

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

DROP TABLE IF EXISTS `t_car`;
CREATE TABLE `t_car`  (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键',
  `car_num` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT '编号',
  `brand` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT '品牌',
  `guide_price` decimal(10, 2) NULL DEFAULT NULL COMMENT '指导价',
  `produce_time` char(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT '生产日期',
  `car_type` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT '汽车类型',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 224 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```

插入表数据

```sql
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (165, '6666', '丰田霸道', 32.00, '2020-11-11', '燃油车');
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (166, '1202', '大众速腾', 30.00, '2020-11-11', '燃油车');
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (167, '1203', '奔驰GLC', 5.00, '2010-12-03', '燃油车');
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (168, '1204', '奥迪Q7', 3.00, '2009-10-11', '燃油车');
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (169, '1205', '朗逸', 4.00, '2001-10-11', '新能源');
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (171, '1207', '奥迪A6', 30.00, '2000-01-02', '燃油车');
INSERT INTO `t_car` (`id`, `car_num`, `brand`, `guide_price`, `produce_time`, `car_type`) VALUES (172, '6666', '丰田霸道', 32.00, '2020-11-11', '燃油车');
```

封装获取SqlSession对象的工具类

```java
public class SqlSessionUtil {
    private static SqlSessionFactory sessionFactory=null;
    private SqlSessionUtil(){

    }

    static {
        try {
            sessionFactory = new SqlSessionFactoryBuilder()
                    .build(Resources.getResourceAsStream("mybatis-config.xml"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static SqlSession getSqlSession(){
        return sessionFactory.openSession(true);
    }
}
```



jdbc.properties

```properties
jdbc.driverClassName=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mybatis
jdbc.username=root
jdbc.password=JXLZZX79
```

CarMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.zzx.mapper.CarMapper">

<resultMap id="carResultMap" type="com.zzx.pojo.Car">
    <id property="id" column="id"/>
    <result property="brand" column="brand"/>
    <result property="carNum" column="car_num"/>
    <result property="carType" column="car_type"/>
    <result property="guidePrice" column="guide_price"/>
    <result property="produceTime" column="produce_time"/>
</resultMap>

<sql id="baseColumn">
    id,car_num,brand,guide_price,produce_time,car_type
</sql>

</mapper>
```

mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    
    <properties resource="jdbc.properties"/>

    <settings>
        <setting name="logImpl" value="STDOUT_LOGGING"/>
    </settings>
    
    <typeAliases>
        <package name="com.zzx.pojo"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driverClassName}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mapper/CarMapper.xml"/>
    </mappers>
</configuration>
```



## 5.2:if标签

需求：多条件查询。

可能的条件包括：品牌（brand）、指导价格（guide_price）、汽车类型（car_type）

- 定义接口

```java
List<Car> selectByMultiCondition(@Param("brand") String brand,
                                 @Param("guidePrice") Double guidePrice,
                                 @Param("carType") String carType);
```

- CarMapper映射配置

```xml
<select id="selectByMultiCondition" resultMap="carResultMap">
    select id,car_num,brand,guide_price,produce_time,car_type from t_car
    where
        <if test="brand != null and brand != ''">
            brand like "%"#{brand}"%"
        </if>
        <if test="guidePrice != null and guidePrice != ''">
            and guide_price >= #{guidePrice}
        </if>
        <if test="carType != null and carType != ''">
            and car_type = #{carType}
        </if>
</select>
```

- 测试类

```java
@Test
public void testIf(){
    CarMapper mapper = SqlSessionUtil.getSqlSession().getMapper(CarMapper.class);
    mapper.selectByMultiCondition("奔驰",26.8,"新能源");
}
```

执行的SQL语句如下，不会报错

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car where brand like "%"?"%" and guide_price >= ? and car_type = ?

奔驰(String), 26.8(Double), 新能源(String)
```

- 测试第一个条件为空，剩下的条件不为空

```java
@Test
public void testIf(){
    CarMapper mapper = SqlSessionUtil.getSqlSession().getMapper(CarMapper.class);
    mapper.selectByMultiCondition(null,26.8,"新能源");
}
```

执行的SQL语句如下，报错

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car where and guide_price >= ? and car_type = ?

Parameters: 26.8(Double), 新能源(String)
```

原因是因为第一个条件不成立，但是后面的成立，则and被添加到了SQL语句，发生了SQL语句错误

这该怎么解决呢？

- 可以where后面添加一个恒成立的条件，在第一个条件加上And

![image-20221001091410971](https://zzx-note.oss-cn-beijing.aliyuncs.com/mybatis/image-20221001091410971.png)

- 解决方式二：使用where标签嵌套在外面，可以智能的处理and是否存在



## 5.3:where标签

where标签的作用：让where子句更加动态智能。

- 所有条件都为空时，where标签保证不会生成where子句。
- 自动去除某些条件**前面**多余的and或or。

继续使用if标签中的需求

修改XML文件

```xml
<select id="selectByMultiCondition" resultMap="carResultMap">
    select id,car_num,brand,guide_price,produce_time,car_type from t_car
    <where>
        <if test="brand != null and brand != ''">
            brand like "%"#{brand}"%"
        </if>

        <if test="guidePrice != null">
            and guide_price >= #{guidePrice}
        </if>

        <if test="carType != null and carType != ''">
            and car_type = #{carType}
        </if>
    </where>
</select>
```

- 测试1

```java
@Test
public void testSelectByMultiConditionWithWhere(){
    CarMapper mapper = SqlSessionUtil.getSqlSession().getMapper(CarMapper.class);
    mapper.selectByMultiCondition("丰田", 20.0, "燃油车");
}
```

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car WHERE brand like "%"?"%" and guide_price >= ? and car_type = ?

Parameters: 丰田(String), 20.0(Double), 燃油车(String)
```

- 测试2

```java
@Test
public void testSelectByMultiConditionWithWhere(){
    CarMapper mapper = SqlSessionUtil.getSqlSession().getMapper(CarMapper.class);
    mapper.selectByMultiCondition(null,null,null);
}
```

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car
```

经过测试：它可以自动去掉前面多余的and，那可以自动去掉前面多余的or吗？

这里就不演示了，它也可以去掉前面多余的or



它可以自动去掉前面多余的and，那可以自动去掉后面多余的and吗？我们随便添加一些在后面

```xml
<select id="selectByMultiCondition" resultMap="carResultMap">
    select id,car_num,brand,guide_price,produce_time,car_type from t_car
    <where>
        <if test="brand != null and brand != ''">
            brand like "%"#{brand}"%" and
        </if>

        <if test="guidePrice != null">
            and guide_price >= #{guidePrice} and
        </if>

        <if test="carType != null and carType != ''">
            and car_type = #{carType}
        </if>
    </where>
</select>
```

```java
@Test
public void testSelectByMultiConditionWithWhere(){
    CarMapper mapper = SqlSessionUtil.getSqlSession().getMapper(CarMapper.class);
    mapper.selectByMultiCondition("丰田", 20.0, "燃油车");
    mapper.selectByMultiCondition(null,null,null);
}
```

结果报错：

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car WHERE brand like "%"?"%" and and guide_price >= ? and and car_type = ?

==> Parameters: 丰田(String), 20.0(Double), 燃油车(String)
```

很显然，后面多余的and是不会被去除的，or同理



## 5.4:trim标签

trim标签的属性：

- prefix：在trim标签中的语句前**添加**内容【加前缀】
- suffix：在trim标签中的语句后**添加**内容【加后缀】
- prefixOverrides：前缀**覆盖掉（去掉）**【删除前缀】
- suffixOverrides：后缀**覆盖掉（去掉）**【删除后缀】



- 接口

```java
/**
* 根据多条件查询Car，使用trim标签
* @param brand
* @param guidePrice
* @param carType
* @return
*/
List<Car> selectByMultiConditionWithTrim(@Param("brand") String brand, 
                                         @Param("guidePrice") Double guidePrice,
                                         @Param("carType") String carType);
```

- CarMapper配置

```xml
<select id="selectByMultiConditionWithTrim" resultType="car">
  select * from t_car
  <!--trim所有内容之前加上前缀where,删除整个trim标签后缀的and或者or-->
  <trim prefix="where" suffixOverrides="and|or">
    <if test="brand != null and brand != ''">
      brand like #{brand}"%"
    </if>
    <if test="guidePrice != null and guidePrice != ''">
      and guide_price >= #{guidePrice}
    </if>
    <if test="carType != null and carType != ''">
      and car_type = #{carType}
    </if>
  </trim>
</select>
```

- 测试类

```java
@Test
public void testTrim(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    Map<String,Object> map=new HashMap<>();
    map.put("brand","奔驰");
    map.put("guidePrice",26.8);
    map.put("carType","燃油车");
    sqlSession.selectList("com.zzx.mapper.CarMapper.selectByMultiConditionWithTrim",map);
}
```

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car where brand like ?"%" and guide_price >= ? and car_type = ?

==> Parameters: 奔驰(String), 26.8(Double), 燃油车(String)
```

- 如果所有条件为空，where会被加上吗？并不会

```sql
select id,car_num,brand,guide_price,produce_time,car_type from t_car
```



## 5.5:set标签

主要使用在update语句当中，用来生成set关键字，同时去掉最后多余的“,”

比如我们只更新提交的不为空的字段，如果提交的数据是空或者""，那么这个字段我们将不更新

```java
/**
* 更新信息，使用set标签
* @param car
* @return
*/
int updateWithSet(Car car);
```

使用这种SQL语句

```xml
<update id="updateWithSet">
    update t_car set car_num=#{carNum},brand=#{brand},
    guide_price=#{guidePrice},produce_time=#{produceTime},cat_type=#{carType}
</update>
```

虽然能成功更新，但是假设前端传过来的参数只有 guidePrice，则其他已有的字段全部会被更新为bull

这个时候就要使用set标签

```xml
<update id="updateWithSet">
    update t_car
    <set>
        <if test="carNum != null and carNum != ''"> car_num=#{carNum}, </if>
        <if test="brand != null and brand != ''"> brand=#{brand}, </if>
        <if test="guidePrice != null"> guide_price=#{guidePrice}, </if>
        <if test="produceTime != null and produceTime != ''"> 
            produce_time=#{produceTime}, </if>
        <if test="carType != null and carType != ''"> car_type=#{carType} </if>
    </set>
    where id=#{id}
</update>
```



## 5.6:choose when otherwise

这三个标签一般是一起使用的

```xml
<choose>
  <when></when>
  <when></when>
  <when></when>
  <otherwise></otherwise>
</choose>
```

类似于

```java
if(){
    
}else if(){
    
}else if(){
    
}else if(){
    
}else{

}
```

需求：先根据品牌查询，如果没有提供品牌，再根据指导价格查询，如果没有提供指导价格，就根据生产日期查询。

```java
/**
* 使用choose when otherwise标签查询
* @param brand
* @param guidePrice
* @param produceTime
* @return
*/
List<Car> selectWithChoose(@Param("brand") String brand, @Param("guidePrice") Double guidePrice, @Param("produceTime") String produceTime);
```

```xml
<select id="selectWithChoose" resultMap="carResultMap">
    select * from t_car
    <where>
        <choose>
            <when test="brand != null and brand != ''"> brand like '%${brand}%'</when>
            <when test="guidePrice != null"> guide_price >= #{guidePrice}</when>
            <otherwise> produce_time >= #{produceTime}</otherwise>
        </choose>
    </where>
</select>
```

测试类

```java
@Test
public void testChoose(){
    //先根据品牌查询，如果没有提供品牌，再根据指导价格查询，如果没有提供指导价格，就根据生产日期查询
    CarMapper mapper = SqlSessionUtil.getSqlSession().getMapper(CarMapper.class);
    mapper.selectWithChoose(null,18.7,"2015-09-24");
    mapper.selectWithChoose("奔驰",18.7,"2015-09-24");
    mapper.selectWithChoose(null,null,null);
}
```



## 5.7:foreach标签

循环数组或集合，动态生成sql，比如这样的SQL：

批量删除：

```sql
delete from t_car where id in(1,2,3);
delete from t_car where id = 1 or id = 2 or id = 3;
```

批量添加：

```sql
insert into t_car values
  (null,'1001','凯美瑞',35.0,'2010-10-11','燃油车'),
  (null,'1002','比亚迪唐',31.0,'2020-11-11','新能源'),
  (null,'1003','比亚迪宋',32.0,'2020-10-11','新能源')
```

### 1:批量删除方式一

- 用in来删除  delete t_car where id in (x,x,x,x)

```java
/**
* 通过foreach完成批量删除
* @param ids
* @return
*/
int deleteBatchByForeach(Long[] ids);
```

```xml
<!--
	collection：集合或数组
	item：集合或数组中的元素
	separator：分隔符
	open：foreach标签中所有内容的开始
	close：foreach标签中所有内容的结束
-->
<delete id="deleteBatchByForeach">
    delete from t_car where id in
        <foreach collection="ids" item="id" separator="," open="(" close=")">
            #{id}
        </foreach>
</delete>
```

发现报错：说collection应该填写array

改进：

```java
/**
 * 通过foreach完成批量删除
 * @param ids
 * @return
 */
int deleteBatchByForeach(@Param("ids") Long[] ids);
```

把接口加上`@Param("ids")`就可以直接在collection里面写 ids 了



### 2:批量删除方式二

- 用or来删除  delete t_car where id = x or id = x or id=x

```java
/**
* 通过foreach完成批量删除
* @param ids
* @return
*/
int deleteBatchByForeach2(@Param("ids") Long[] ids);
```

```xml
<delete id="deleteBatchByForeach2">
  delete from t_car where
  <foreach collection="ids" item="id" separator="or">
    id = #{id}
  </foreach>
</delete>
```



### 3:批量添加

```java
/**
* 批量添加，使用foreach标签
* @param cars
* @return
*/
int insertBatchByForeach(@Param("cars") List<Car> cars);
```

```java
<insert id="insertBatchByForeach">
  insert into t_car values 
  <foreach collection="cars" item="car" separator=",">
    (null,#{car.carNum},#{car.brand},#{car.guidePrice},#{car.produceTime},#{car.carType})
  </foreach>
</insert>
```



## 5.8:ResultMap

resultMap 需要一个ID来标识，type就是这个实体类

主键字段使用 id

其他字段使用 result

具体的匹配就是 实体类的 `propert` 对应 数据库的 `column`

```xml
<resultMap id="carResultMap" type="com.zzx.pojo.Car">
    <id property="id" column="id"/>
    <result property="brand" column="brand"/>
    <result property="carNum" column="car_num"/>
    <result property="carType" column="car_type"/>
    <result property="guidePrice" column="guide_price"/>
    <result property="produceTime" column="produce_time"/>
</resultMap>
```

使用ResultMap

```xml
<select id="selectByMultiCondition" resultMap="carResultMap">
    select id,car_num,brand,guide_price,produce_time,car_type from t_car
    where
        <if test="brand != null and brand != ''">
            brand like "%"#{brand}"%" and
        </if>
        <if test="guidePrice != null and guidePrice != ''">
            guide_price >= #{guidePrice} and
        </if>
        <if test="carType != null and carType != ''">
            car_type = #{carType}
        </if>
</select>
```

## 5.9:sql标签与include标签

sql标签用来声明sql片段

include标签用来将声明的sql片段包含到某个sql语句当中

作用：代码复用。易维护。



定义SQL：<sql id="xxx"> 需要的SQL字段 </sql>

```sql
<sql id="carCols">
	id,car_num carNum,brand,guide_price guidePrice,produce_time produceTime,car_typecarType
</sql>
```



使用SQL：使用<include refid= "定义的SQL的ID">

```sql
<select id="selectAllRetMap" resultType="map">
  select <include refid="carCols"/> from t_car
</select>
```



# 6:PageHelper

## 6.1:limit分页

mysql的limit后面两个数字：

- 第一个数字：startIndex（起始下标。下标从0开始。）
- 第二个数字：pageSize（每页显示的记录条数）

假设已知页码pageNum，还有每页显示的记录条数pageSize，第一个数字可以动态的获取吗？

- startIndex = (pageNum - 1) * pageSize

所以，标准通用的mysql分页SQL：

```sql
select * from tableName ...... limit (pageNum - 1) * pageSize, pageSize
```

使用mybatis应该怎么做？



## 6.2:自己分页

- 不使用插件，自己写

```java
List<Car> selectAllByPage(Integer pageNo,Integer pageSize);
```

xml配置

```xml
<mapper namespace="com.zzx.mapper.CarMapper">
    <select id="selectAllByPage" resultType="com.zzx.domain.Car">
        select * from t_car limit #{pageNo},#{pageSize}
    </select>
</mapper>
```

注意：传递参数的时候是 `(pageNo-1)*pageSize , pageSize`

测试类

```java
@Test
public void test01(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    mapper.selectAllByPage(1,5);
}
```

```sql
==> Preparing: select * from t_car limit ?,?
==> Parameters: 1(Integer), 5(Integer)
```

获取数据不难，难的是获取分页相关的数据比较难【总条数，上下翻页等信息】

可以借助mybatis的PageHelper插件



## 6.3:使用插件

使用PageHelper插件进行分页，更加的便捷。

### 1:引入依赖

```xml
<dependency>
  <groupId>com.github.pagehelper</groupId>
  <artifactId>pagehelper</artifactId>
  <version>5.3.1</version>
</dependency>
```



### 2:配置插件

mybatis-config.xml文件中配置插件

typeAliases标签下面进行配置：

```xml
<plugins>
  <plugin interceptor="com.github.pagehelper.PageInterceptor"></plugin>
</plugins>
```



### 3:使用插件

- 接口

```java
List<Car> selectAll();
```

- Mapper配置

```xml
<select id="selectAll" resultType="com.zzx.domain.Car">
    select * from t_car
</select>
```

关键点：

- 在查询语句之前开启分页功能。
- 在查询语句之后封装PageInfo对象（PageInfo对象将来会存储到request域当中。在页面上展示。）
- PageInfo对象有很多方法，如下所示

![image-20221002211022729](https://zzx-note.oss-cn-beijing.aliyuncs.com/mybatis/image-20221002211022729.png)

演示：

```java
@Test
public void test02(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    PageHelper.startPage(3,6);
    List<Car> cars = mapper.selectAll();
    PageInfo<Car> carPageInfo = new PageInfo<>(cars);
    System.out.println("数据总数:"+carPageInfo.getTotal());
    System.out.println("总页数:"+carPageInfo.getPages());
    System.out.println("当前页大小:"+carPageInfo.getSize());
}
```

启动的时候出现：

![image-20221002211134274](https://zzx-note.oss-cn-beijing.aliyuncs.com/mybatis/image-20221002211134274.png)

表明是使用了拦截器，对之前查到的结果执行了拦截，动态添加了 `limit`

```sql
==>  Preparing: SELECT count(0) FROM t_car
==> Parameters: 
<==    Columns: count(0)
<==        Row: 15
<==      Total: 1
==>  Preparing: select * from t_car LIMIT ?, ?
==> Parameters: 12(Long), 6(Integer)
```

# 7:注解式开发

mybatis中也提供了注解式开发方式，采用注解可以减少Sql映射文件的配置。

当然，使用注解式开发的话，sql语句是写在java程序中的，这种方式也会给sql语句的维护带来成本



官方是这么说的：

使用注解来映射简单语句会使代码显得更加简洁，但对于稍微复杂一点的语句，Java 注解不仅力不从心

还会让你本就复杂的 SQL 语句更加混乱不堪。 

因此，如果你需要做一些很复杂的操作，最好用 XML 来映射语句

原则：简单sql可以注解。复杂sql使用xml



## 7.1:@Insert

```java
public interface CarMapper {

    @Insert(value="insert into t_car values
           (null,#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})")
    int insert(Car car);
}
```

XML配置

```java
@Test
public void test03(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    Car car = new Car();
    car.setCarNum("1123");
    car.setCarType("汽油车");
    car.setBrand("奔驰");
    car.setGuidePrice(new BigDecimal(23.56));
    car.setProduceTime("2021-09-06");
    mapper.insert(car);
}
```

执行成功没有问题

同理：

## 7.2:@Delete

```java
@Delete("delete from t_car where id = #{id}")
int deleteById(Long id);
```



## 7.3:@Update

```java
@Update("update t_car set car_num=#{carNum},brand=#{brand},guide_price=#{guidePrice},produce_time=#{produceTime},car_type=#{carType} where id=#{id}")
int update(Car car);
```



## 7.4:@Select

@Select相对有点不一样，可以使用@Results来配置结果集

```java
@Select("select * from t_car where id = #{id}")
@Results({
    @Result(column = "id", property = "id", id = true),
    @Result(column = "car_num", property = "carNum"),
    @Result(column = "brand", property = "brand"),
    @Result(column = "guide_price", property = "guidePrice"),
    @Result(column = "produce_time", property = "produceTime"),
    @Result(column = "car_type", property = "carType")
})
Car selectById(Long id);
```

```java
@Test
public void test04(){
    SqlSession sqlSession = SqlSessionUtil.getSqlSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    Car car = mapper.selectById(210L);
    System.out.println(car);
}
```

```sql
==> Preparing: select * from t_car where id = ?
==> Parameters: 210(Long)
```

实际上和XML配置结果集是一样的，也是把查询到的结果的 `column` 赋值给 `@Result` 的 `property` 属性