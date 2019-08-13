---
title: Hibernate 关联映射：无连接表双向1-N
date: 2018-03-24 16:10:30
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79678491]( https://blog.csdn.net/abc123lzf/article/details/79678491)   
  ## 一、简介

 双向1-N关联应当尽量不要让1的一端控制关联关系，而使用N的一端控制关系。双向的1-N关联与N-1关联是完全相同的两种情形，两端都需要增加对关联属性的访问，N的一端增加引用到关联实体的属性，1的一端增加集合属性，集合元素为关联实体。   
 对于大部分1-N关联来说，使用无连接表的策略即可。

 
## 二、实例演示：

 
#### （1）建立两个表：classroom表和person表（MySQL）

 表名：person   
 ![这里写图片描述](https://img-blog.csdn.net/20180324153151521?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
 建表语句：

 
```
CREATE TABLE `person` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(20) NOT NULL,
    `age` int(11) DEFAULT NULL,
    `class_id` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `class_id` (`class_id`),
    CONSTRAINT `person_ibfk_1` FOREIGN KEY (`class_id`) REFERENCES      `classroom` (`classroom_id`) ON DELETE SET NULL ON UPDATE CASCADE
);
```
 表名：classroom   
 ![这里写图片描述](https://img-blog.csdn.net/20180324153807278?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
 建表语句：

 
```
CREATE TABLE `classroom` (
    `classroom_id` int(11) NOT NULL AUTO_INCREMENT,
    `classroom_name` varchar(50) NOT NULL,
    PRIMARY KEY (`classroom_id`)
);
```
 
#### （2）建立两个类：ClassRoom类和Person类

 ClassRoom.java

 
```
@Entity
@Table(name="classroom")
public class ClassRoom
{
    @Id @Column(name="classroom_id")
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Integer id;

    @Column(name="classroom_name")
    private String classRoomName;

    //定义该ClassRoom实体所有关联的Person实体
    //mappedBy用来表面该ClassRoom实体不控制关联关系
    @OneToMany(targetEntity=Person.class,mappedBy="classRoom")
    private Set<Person> personSet = new HashSet<>();

    public ClassRoom() {}

    public ClassRoom(String classRoomName)
    {
        this.classRoomName = classRoomName;
    }

    public Set<Person> getPersonSet()
    {
        return personSet;
    }

    public void setPersonSet(Set<Person> personSet) 
    {
        this.personSet = personSet;
    }

    public String getClassRoomName() 
    {
        return classRoomName;
    }

    public void setClassRoomName(String classRoomName) 
    {
        this.classRoomName = classRoomName;
    }
}

```
 上面的ClassRoom类增加一个set集合属性，用于记录它关联的一系列Person实体，使用@OneToMany修饰该集合属性，表明此处是1-N关联。另外，Person端只需增加一个ClassRoom属性，表明ClassRoom和Person属性存在1-N的关联关系。

 Person.java

 
```
@Entity
@Table(name="person")
public class Person
{
    @Id @Column(name="id")
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Integer id;

    private String name;
    private Integer age;

    @ManyToOne(targetEntity=ClassRoom.class)
    //定义名为class_id外键列，该外键列引用classroom表的classroom_id列
    @JoinColumn(name="class_id",referencedColumnName="classroom_id",nullable=false)
    private ClassRoom classRoom;

    public Person() {}

    public Person(String name,Integer age)
    {
        this.name = name;
        this.age = age;
    }

    public Person(String name,Integer age,ClassRoom classRoom) 
    {
        this.name = name;
        this.age = age;
        this.classRoom = classRoom;
    }

    public String getName() 
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }

    public Integer getAge() 
    {
        return age;
    }

    public void setAge(Integer age) 
    {
        this.age = age;
    }

    public void setClassRoom(ClassRoom classRoom)
    {
        this.classRoom = classRoom;
    }

    public ClassRoom getClassRoom()
    {
        return classRoom;
    }
}
```
 上面注解使用了@ManyToOne代表关联实体的属性，也使用了@JoinColumn映射外键列，该注解将会控制在person表中增加名为class_id的外键列，意味着person表将作为从表使用。

 
#### （3）添加数据

 运行以下程序：

 
```
public class Main
{
    public static void main(String[] args)
    {
        Session session = HibernateUtil.currentSession();
        Transaction tx = session.beginTransaction();

        Person p1 = new Person("Li",19);
        Person p2 = new Person("Zhang",20);
        ClassRoom classRoom = new ClassRoom("东阶教室");

        session.persist(classRoom);

        p1.setClassRoom(classRoom);
        session.persist(p1);

        p2.setClassRoom(classRoom);
        session.persist(p2);

        tx.commit();
        HibernateUtil.closeSession();
    }
}
```
 HibernateUtil参考：[https://blog.csdn.net/vipmao/article/details/51340525](https://blog.csdn.net/vipmao/article/details/51340525)   
 运行后表中数据为：   
 ![这里写图片描述](https://img-blog.csdn.net/20180324160435201?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 ![这里写图片描述](https://img-blog.csdn.net/20180324160459811?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 
#### （4）查找示例

 
```
public class Main
{
    public static void main(String[] args)
    {
        Session session = HibernateUtil.currentSession();
        Transaction tx = session.beginTransaction();

        //获取所有ClassRoom对象
        List list = session.createQuery("from ClassRoom as a").list();
        for(Object o1 : list)
        {
            ClassRoom cr = (ClassRoom)o1;
            //获取这个ClassRoom存储有对应person的set属性
            Set set = cr.getPersonSet();
            //输出Person的Name属性
            for(Object o2 : set)
            {
                Person p = (Person)o2;
                System.out.println(p.getName());
            }
        }

        tx.commit();
        HibernateUtil.closeSession();
    }
}
```
 控制台将会输出：   
 Li   
 Zhang

   
  