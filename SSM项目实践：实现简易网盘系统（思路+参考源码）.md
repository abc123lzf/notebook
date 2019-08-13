---
title: SSM项目实践：实现简易网盘系统（思路+参考源码）
date: 2018-04-17 19:23:51
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79977628]( https://blog.csdn.net/abc123lzf/article/details/79977628)   
  ### 一、项目主要功能

 1、类似于百度云，用户可以在网盘中新建多层文件夹，并可以上传文件、下载文件、删除文件、删除文件夹（里面所有文件也会随之删除）   
 2、提供文件分类功能，可根据后缀名将文件分成视频、图片、音乐等。   
 3、用户可以分享文件，分享的文件可以在分享区中看到并下载。

 
#### 结构：

 ![这里写图片描述](https://img-blog.csdn.net/20180417190943889?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 
#### 效果：

 ![这里写图片描述](https://img-blog.csdn.net/20180417171503726?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 ![这里写图片描述](https://img-blog.csdn.net/20180417171722844?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 ![这里写图片描述](https://img-blog.csdn.net/20180417171930406?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 
### 二、数据表设计

 这里采用MySQL数据库   
 为了实现功能，我们需要三个数据表：记录用户信息、文件夹信息和文件信息的数据表   
 user_inf表   
 ![这里写图片描述](https://img-blog.csdn.net/20180417161122080?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
 其中，status用于记录用户是否为新注册用户

 dir_inf表   
 ![这里写图片描述](https://img-blog.csdn.net/20180417161414254?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
 parent_dir用于记录父文件夹id，如果该文件夹在根目录则为null。   
 dir_user_id记录该文件夹的上传用户，该列为外键列，与user_inf的user_id列相关联。   
 dir_path用于记录文件夹绝对路径，主要为了方便进行文件下载。

 file_inf表   
 ![这里写图片描述](https://img-blog.csdn.net/20180417161835061?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
 file_status记录该文件是否公开。   
 file_dir_id记录该文件的所属文件夹id，该列为外键列，与dir_inf表的dir_id列相关联。   
 file_upload_user_id记录该文件上传用户id，为外键列，与user_inf表的user_id列相关联。

 
### 三、项目关键点分析：

 
#### 1、类的设置

 主要分为三个类型：控制器类，实体类和系统工具类

 
##### （1）控制器类

 包含视图控制器类（负责管理主要URL并返回WEB-INF中的JSP视图文件）、文件管理器类（主要作用为对网盘数据的各种操作）、登录和注册类（不再赘述）

 
##### （2）实体类

 实体类对应上述三个表：User类、CloudFile类、Folder类，每个类应建立对应的Mapper接口和XML文件。

 
##### （3）工具类

 MyBatisUtil类：使用静态初始化块读取mybatis-config.xml配置文件，并提供一个返回SqlSessionFactory的静态方法，避免每次需要调用数据库时重复读取xml配置文件（也可采用Spring的单例模式）   
 其他工具类：记录用户文件存放根目录的字符串，获取系统时间的静态方法、对包含特殊字符的SQL语句处理的静态方法等等。

 
#### 2、用户注册与登录

 新用户在注册后，往user_inf表插入数据，并将status状态设为新注册状态。第一次登录时服务器在存放用户文件的目录新建一个文件夹，并命名为该用户的用户名。登录时需将用户对象存入session中。

 
#### 3、用户文件存放路径：

 可以在Tomcat根目录或者其他路径新建一个文件夹命名为UserData，UserData中每个文件夹对应一个用户，文件夹名为用户的username，用户自己新建的文件夹和上传的文件全部存储在该文件夹中。

 
#### 4、文件管理JSP页面：

 在视图控制器中，首先需要获取session中的user对象，根据该user对象查找出根目录的id，通过该id值查找根目录下的文件夹和文件，封装成List并存入request中，通过循环标签体循环打印出表格，并且将下面所有的功能对应的URL引入到表格中，参数由EL表达式获取。用户进入一个子文件夹后需要记录父文件夹id并存入request中，以便用户可以退回上一级目录。

 
#### 5、用户新建文件夹：

 定义一个专门用来新建文件夹的URL，该URL需要两个参数：文件夹名和父文件夹id，文件夹名由用户自己填写，父文件夹id由JSP页面获取request属性并将其隐藏在表单中。前台页面及后台需要检查用户是否输入了文件夹名以及文件夹名是否合法。   
 后台处理过程：   
 （1）根据父文件夹id在数据库查找对应的父文件夹信息并封装成对象。   
 （2）新建一个文件夹对象，设置该对象的父文件夹属性、用户属性等   
 （3）获取父文件夹对象中的绝对路径属性，通过File类的mkdirs()方法新建文件夹   
 （4）将新文件夹对象插入到数据库中

 
#### 6、上传文件（支持同时选择多个文件）：

 前台JSP页面建立一个表单，表单元素为文件上传框并支持多文件选择、父文件夹ID由JSP页面获取request属性设置并隐藏在表单中。后台采用MultipartFile[]来接收文件。   
 后台处理过程：   
 （1）检查用户是否选择了文件   
 （2）根据父文件夹id在数据库查找对应的父文件夹信息并封装成对象。   
 （3）遍历MultipartFile[]数组，新建一个文件对象，通过MultipartFile接口提供的getOriginalFilename等方法设置该文件对象属性并插入到文件数据表中   
 （4）通过父文件夹对象获取绝对路径，通过MultipartFile接口提供的transferTo方法在服务器建立该文件。   
 删除文件和上传文件类似，此处不再赘述

 
#### 7、删除文件夹

 这个地方难点在于如果包含多层子文件夹和子文件该如何删除，我的思路是可以采用SQL的模糊查询来完成：因为数据表中存有绝对路径，所以在查找一个文件夹中所有的子文件夹可以通过like ‘${path}%’来完成，然后通过查找出的子文件夹id查找其中的文件，删除即可。对于实际存储在服务器上的文件夹而言，需要用到递归删除，关于递归删除可以参考这篇博客：[https://www.cnblogs.com/zhenyu-go/p/5554979.html](https://www.cnblogs.com/zhenyu-go/p/5554979.html)。   
 后台只需接收一个参数：需要删除的文件夹的id。   
 具体流程：   
 （1）根据目标文件夹id查找并封装成该文件夹对象   
 （2）先通过该文件夹id查找出该文件夹下的文件并删除（不包含子文件夹中的文件）   
 （3）通过该文件夹的绝对路径模糊查询出所有子文件夹的List   
 （4）遍历该list，一一根据该list元素中文件夹的id删除文件夹和其中的文件   
 （5）将目标文件夹删除   
 （6）递归删除实际文件夹（不能通过File类的delete直接删除）

 
#### 8、文件分享和文件分类

 文件分享：通过文件id将status设为公开状态，在文件分享JSP页面循环打印出所有文件即可。这个地方比较简单   
 文件分类可通过文件后缀名查找出来。

 
### 四、本示例参考源码:

 代码写得可能有些乱，敬请各位看官谅解，如果有更好的思路或建议欢迎留言。   
 如果需要运行的话可能里面有很多配置要改，放出代码只是提供一个思路：   
 链接: [https://pan.baidu.com/s/1RipG9h6YY4OxM5JIB_kr3w](https://pan.baidu.com/s/1RipG9h6YY4OxM5JIB_kr3w) 密码: t1gb

   
  