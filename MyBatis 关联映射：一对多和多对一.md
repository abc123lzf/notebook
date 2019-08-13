---
title: MyBatis 关联映射：一对多和多对一
date: 2018-04-08 12:16:55
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79845791]( https://blog.csdn.net/abc123lzf/article/details/79845791)   
  ### 1、前言

 一对多在Web项目中是一种很常见的关系。例如：一个班级对应多个学生，一个学生仅属于一个班级。数据库中一对多关系通常使用主外键关联，外键列应在多方。下面我们通过代码示例来演示如何使用MyBatis处理这种关系。

 
### 2、实例：教室和学生

 
#### （1）MySQL部分

 为了方便描述，我们建立两个表：student_inf表和classroom_inf表   
 classroom_inf表：

 
```
CREATE TABLE `classroom_inf` (
  `classroom_id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  PRIMARY KEY (`classroom_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8
```
 student_inf表:

 
```
CREATE TABLE `student_inf` (
  `student_id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  `age` int(11) DEFAULT NULL,
  `classroom_id` int(11) NOT NULL,
  PRIMARY KEY (`student_id`),
  KEY `classroom_id` (`classroom_id`),
  CONSTRAINT `student_inf_ibfk_1` FOREIGN KEY (`classroom_id`) REFERENCES `classroom_inf` (`classroom_id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8
```
 然后往表中插入示例数据，如图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180407221549482?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 ![这里写图片描述](https://img-blog.csdn.net/20180407221707480?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 
--------
 
#### （2）创建实体对象

 首先是Student类   
 由于Student类属于多的一方，我们应当增加一个classRoom属性，这个属性的类型应当是1的一方，即ClassRoom类型。   
 代码如下：   
 com/demo/entity/Student.java

 
```
public class Student
{
    private Integer id;
    private String name;
    private Integer age;
    //每个人对应一个教室
    private ClassRoom classRoom;

    public Student(){}

    public Student(String name,Integer age)
    {
        this.name = name;
        this.age = age;
    }

    //省略setter、getter方法
}
```
 然后是ClassRoom类   
 由于ClassRoom属于1的一方，即一个教室可以有多个学生，所以我们应当定义一个students属性，该属性为一个List集合，用来映射这种关系。   
 代码如下：   
 com/demo/entity/ClassRoom.java

 
```
public class ClassRoom 
{
    private Integer id;
    private String name;

    private List<Student> students;

    public ClassRoom() {}

    public ClassRoom(String name) 
    {
        this.name = name;
    }

    //省略getter、setter方法
}
```
 
--------
 
#### （3）建立XML配置文件

 在多的一方，应当使用关联映射<association…/>，<association…/>元素的解释如下：   
 1、property：表示返回类型Student属性名classRoom   
 2、javaType：表示该属性对应的名称，即ClassRoom   
 这里需要注意的是：由于Student表和ClassRoom表存在同列名：name，为了防止混淆，在select语句中我们需要给其中一个表的name列取别名。否则在实际查找中当我们想输出一个student的name时却发现输出的是classroom的name。   
 下面看源码：   
   
 com/demo/entity/StudentMapper.xml

 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--namespace为对应的Mapper接口路径-->
<mapper namespace="com.demo.entity.StudentMapper">  

    <resultMap id="studentResultMap" type="com.demo.entity.Student">
        <id property="id" column="student_id" />
        <!--表中的实际列名为name，为了防止混淆这里取student_name，同时也要在下面的SQL语句中声明-->
        <result property="name" column="student_name" />
        <result property="age" column="age" />
        <association property="classRoom" javaType="com.demo.entity.ClassRoom">
            <id property="id" column="classroom_id" />
            <result property="name" column="name" />
        </association>
    </resultMap>

    <!--通过学生id查找这个学生的信息-->
    <select id="selectStudentById" parameterType="int" resultMap="studentResultMap">
        SELECT
             c.*,
             s.*,
             s.name student_name <!--给student_inf表的name列名取别名-->
        FROM classroom_inf c,student_inf s
        WHERE c.classroom_id = s.classroom_id AND s.student_id = #{id}
    </select>

    <!--查找所有学生信息-->
    <select id="selectStudent" resultMap="studentResultMap">
        SELECT 
            c.*,
            s.*,
            s.name student_name
        FROM classroom_inf c,student_inf s
        WHERE c.classroom_id = s.classroom_id
    </select>

</mapper>
```
 在1的一方，应当在resultMap元素中使用collection元素来映射这种关系，<collection…/>元素解释如下：   
 1、property：表示返回类型List<Student>属性名students   
 2、ofType：集合中元素的类型。   
 com/demo/entity/ClassRoomMapper.xml

 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="com.demo.entity.ClassRoomMapper">    

    <resultMap id="classRoomMap" type="com.demo.entity.ClassRoom">
        <id property="id" column="classroom_id" />
        <result property="name" column="name" />

        <collection property="students" ofType="com.demo.entity.Student">
            <id property="id" column="student_id" />
            <!--这里同样需要取别名student_name-->
            <result property="name" column="student_name" />
            <result property="age" column="age" />
        </collection>

    </resultMap>

    <!--查找所有教室信息和对应的学生信息-->
    <select id="selectClassRoom" resultMap="classRoomMap">
        SELECT 
            c.*,
            s.*,
            s.name student_name
        FROM classroom_inf c, student_inf s
        WHERE c.classroom_id = s.classroom_id
    </select>

    <!--根据id查找所有教室信息和对应的学生信息-->
    <select id="selectClassRoomById" resultMap="classRoomMap" parameterType="int">
        SELECT 
            c.*,
            s.*,
            s.name student_name
        FROM classroom_inf c, student_inf s
        WHERE c.classroom_id = s.classroom_id AND c.classroom_id = #{id}
    </select>

</mapper>
```
 
--------
 
#### （4）创建对应的Mapper接口

 Mapper接口中的方法名、参数、返回值必须和XML配置文件一致

 
```
package com.demo.entity.StudentMapper
public interface StudentMapper {
    public Student selectStudentById(Integer id);
    public List<Student> selectStudent();
}
```
 
```
package com.demo.entity.ClassRoomMapper
public interface ClassRoomMapper {
    public List<ClassRoom> selectClassRoom();
    public ClassRoom selectClassRoomById(Integer id);
}
```
 
#### （5）测试类

 建议使用一个工具类读取XML配置文件，并直接返回SqlSessionFactory，避免每次连接数据库时反复读取XML文件

 
```
public final class MyBatisUtil {
    private static SqlSessionFactory sqlSessionFactory;
    static {
        try {
            InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static SqlSessionFactory getSqlSessionFactory() {
        return sqlSessionFactory;
    }
    private MyBatisUtil() { }
}
```
 com/demo/Main.java

 
```
public class Main 
{
    public static void main(String[] args) throws IOException
    {
        try (SqlSession sqlSession = MyBatisUtil.getSqlSessionFactory().openSession()) {
            ClassRoomMapper classRoomMapper = sqlSession.getMapper(ClassRoomMapper.class);  
            //查找所有教室信息
            List<ClassRoom> classList = classRoomMapper.selectClassRoom();
            //循环输出内容
            for(ClassRoom classRoom : classList)
            {
                System.out.println(classRoom.getName());
                //获取每个教室对应的学生
                List<Student> students = classRoom.getStudents();
                for(Student student : students)
                    System.out.println(student.getName() + " " + student.getAge());
            }

            StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
            //查找所有学生信息
            List<Student> studentList = studentMapper.selectStudent();
            //循环输出学生信息
            for(Student student : studentList)
            {
                System.out.println(student.getName() + " " + student.getAge() + " " +
                     student.getClassRoom().getName());
            }
        }
    }
}
```
 输出内容：

 
```
1401教室
李同学 17
张同学 18
1402教室
刘同学 19
1403教室
陈同学 20
李同学 17 1401教室
张同学 18 1401教室
刘同学 19 1402教室
陈同学 20 1403教室
```
   
  