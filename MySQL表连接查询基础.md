---
title: MySQL表连接查询基础
date: 2018-03-23 19:41:45
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79671202]( https://blog.csdn.net/abc123lzf/article/details/79671202)   
  ## 一、表连接查询的用途

 
##### 当需要同时显示多个表的字段时，就可以用表连接来实现这样的功能。表连接分为内连接和外连接。内连接仅选出两张表中互相匹配的记录，而外连接会选出其他不匹配的记录。

 
## 二、示例

 
##### 以下面两张表为示范：

 表名:teacher_table   
 ![这里写图片描述](https://img-blog.csdn.net/20180323185039531?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 建表语句：

 
```
create table teacher_table(
    teacher_id int not null auto_increment primary key,
    teacher_name varchar(20),
    phone_number varchar(20),
);
```
 表名:student_table   
 ![这里写图片描述](https://img-blog.csdn.net/20180323185738328?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
 建表语句：

 
```
create table student_table(
    id int not null auto_increment primary key,
    name varchar(20),
    teacher int,
    foreign key(teacher) references teacher_table(teacher_id) on 
        delete set null on update cascade
    );
```
 
### 1、内连接示例

 表连接查询语句语法如下：

 
```
select column1,column2... 
from table1,table2...
[where condition];
```
 其中，column为待查询的列名，table为需要引用的表。   
 如果两个表中有相同的列名，则需要在这些同名列之间使用表别名前缀或表名前缀作为限制，以免混淆。

 
#### 例：

 
```
select 
    name,teacher_name 
from 
    teacher_table t,student_table s 
where 
    s.teacher = t.teacher_id;
```
 执行结果：   
 ![这里写图片描述](https://img-blog.csdn.net/20180323191222302?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 
### 2、外连接

 
##### 外连接分为左外连接和右外连接，具体定义为：

 左连接：包含所有的左边表中的记录甚至是右边表中没有和它匹配的记录。   
 右连接：包含所有的右边表中的记录甚至是左边表中没有和它匹配的记录。

 
#### 左连接查询示例：

 
```
select 
    teacher_name,name 
from 
    teacher_table t left join student_table s 
on 
    t.teacher_id = s.teacher;
```
 查询结果：   
 ![这里写图片描述](https://img-blog.csdn.net/20180323192331071?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 右连接和左连接类似，两者可以互相转换，例如上面的例子可以改写为：

 
```
select 
    teacher_name,name 
from 
    student_table s  right join teacher_table t
on 
     s.teacher = t.teacher_id;
```
 查询结果同上

   
  