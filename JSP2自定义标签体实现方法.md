---
title: JSP2自定义标签体实现方法
date: 2017-12-26 18:26:25
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/78905215]( https://blog.csdn.net/abc123lzf/article/details/78905215)   
  ## 一、自定义标签库的作用

 自定义标签库是一种很优秀的表现层组件技术。通过使用标签库，可以在简单的标签中封装复杂的功能，取代页面中“丑陋”的JSP脚本，提高代码的易读性，使维护和调试更加方便。

 
## 二、开发标签库简要步骤

 
### （1）开发自定义标签处理类

 自定义标签类应当继承一个父类：javax.servlet.jsp.tagext.SimpleTagSupport，自定义标签类中如果包含属性，则需要为每个属性提供setter和getter方法。每个标签类需要重写doTag()方法。   
 如下列标签输出HelloWorld.

 
```
package taglib;
public class firstTag extends SimpleTagSupport
{
    public void doTag() throws Exception
    {
        this.getJspContext().getOut().write("Hello,world!");
    }
}
```
 
### （2）建立一个*.tld的标签库定义文件

 每个标签库定义文件包含一个标签库，一个tld文件可以包含多个标签。标签库定义文件的根元素是taglib，可以包含多个tag子元素，每个tag元素对应一个标签。   
 tld文件应放在web-inf文件夹或web-inf文件夹的任意子路径下，服务器会自动加载该文件。   
 下面是一个tld文件示范，定义了上述helloworld标签体：

 
```
<?xml version="1.0" encoding="UTF-8"?>
<taglib xmlns="http://java.sun.com/xml/ns/j2ee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-jsptaglibrary_2_0.xsd"
    version="2.0">
    <tlib-version>1.0</tlib-version>
    <short-name>mytaglib</short-name>
    <uri>http://tomcat.apache.org/jsp2-example-taglib</uri>

    <tag>
        <name>helloworld</name>
        <tag-class>taglib.firstTag</tag-class>
        <body-content>empty</body-content>
    </tag>

</taglib>
```
 taglib有三个子元素：   
 （1）tlib-version：指定该标签库实现版本。   
 （2）short-name：标签库默认短名，无太大作用。   
 （3）uri：指定标签库URI，即该标签库唯一标识，JSP页面中使用标签库就是根据该URI定义的。   
 tag中的子元素有：   
 （1）name：该标签名称，JSP页面中需要根据该名称来使用此标签，为必须属性。   
 （2）tag-class：该标签的实现类，决定该标签由哪个类来进行处理。   
 （3）body-content：指定该标签体内容，子元素可以有以下几个：   
 1、tagdependent：指定标签处理类自己负责处理标签体。   
 2、empty：指定该标签体只能作为空标签来使用。   
 3、scriptless：指定该标签体可以是静态HTML元素、表达式语言，但不允许出现JSP脚本。   
 4、JSP：指定该标签体可以使用JSP脚本。   
 （4）dynamic-attributes：指定该标签体是否支持动态属性。

 
### （3）在JSP页面中使用自定义标签

 
#### 1、导入标签库

 导入标签库语法如下：

 
```
<%@ taglib uri="TagLibUri" prefix="tagPrefix"%>
```
 其中，uri属性指定标签库的URI，prefix指定标签库前缀，可以自定义一个名称。   
 使用标签语法为：

 
```
<tagPrefix:tagName tagAttribute="tagValue"... />
```
 或者

 
```
<tagPrefix:tagName tagAttribute="tagValue"... >
<tagBody />
</tagPrefix:tagName>
```
 以上内容节选自《轻量级JavaEE：企业应用实战（第四版）》

 
## 三、实例：迭代器标签体

 
#### （1）定义一个标签处理类

 
```
package taglib;

import java.io.IOException;
import java.io.Writer;
import java.util.ArrayList;
import java.util.Collection;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.PageContext;
import javax.servlet.jsp.tagext.SimpleTagSupport;

public class IteratorTag extends SimpleTagSupport
{
    private String collection; //标签属性：集合名称
    private String item; //标签属性：为集合元素指定的名称

    //省略collection、item的setter、getter方法

    public void doTag() throws JspException, IOException
    {
        Collection itemList = (Collection)getJspContext().getAttribute(collection);

        for(Object s : itemList)
        {
            getJspContext().setAttribute(item, s);
            getJspBody().invoke(null);
        }
    }
}
```
 
#### (2)定义一个需要输出的类

 以Blog类作为示范：

 
```
public class Blog
{
    private Integer id;
    private String title,author,time,content;

    public Blog(Integer id,String title,String content,String time,String author)
    {
        this.setId(id);
        this.setTitle(title);
        this.setContent(content);
        this.setTime(time);
        this.setAuthor(author);
    }
    //省略所有getter、setter方法
}
```
 
### （3）获取Blog实例并存储在集合中，并将该集合对象存入session中（略）

 
### （4）应用于JSP页面

 
```
<%@ page language="java" contentType="text/html; charset=utf-8"
    pageEncoding="utf-8"%>
<%@ taglib uri="http://tomcat.apache.org/jsp2-example-taglib" prefix="mytag"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>博客列表</title>
</head>
<body>
    <!--获取session中的List<Blog>-->
    <% pageContext.setAttribute("blogLists", session.getAttribute("blogList")); %>
    <!--输出List中Blog实例信息的表格-->
    <table border="1">
        <tr>
            <th>ID</th>
            <th>标题</th>
            <th>作者</th>
            <th>发布时间</th>
        </tr>
        <mytag:iterator collection="blogLists" item="blog">
            <tr>
                <td>${pageScope.blog.id}</td>
                <td>${pageScope.blog.title }</td>
                <td>${pageScope.blog.author }</td>
                <td>${pageScope.blog.time }</td>
            </tr>
        </mytag:iterator>
    </table>
</body>
</html>
```
   
  