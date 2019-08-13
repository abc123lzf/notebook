---
title: SSM框架实现Ajax发送请求并处理JSON格式返回数据
date: 2018-05-08 15:37:21
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/80240670]( https://blog.csdn.net/abc123lzf/article/details/80240670)   
  ### 一、包及相关库配置

 实现JSON必须要引入相关包，下载地址可参考这篇博客：   
 [https://blog.csdn.net/salonzhou/article/details/36682981](https://blog.csdn.net/salonzhou/article/details/36682981)   
 另外也要在JSP或HTML页面引入jQuery

 
### 二、Spring基本配置（只列出必须bean）

 
```
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping" />

<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">   
    <property name="messageConverters">  
        <list>  
            <ref bean="mappingJacksonHttpMessageConverter" />  
        </list>  
    </property>  
</bean>   

<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" />  

<bean id="viewResolver"  class="org.springframework.web.servlet.view.UrlBasedViewResolver">  
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView" />  
    <property name="prefix" value="/WEB-INF/view/" />  
    <property name="suffix" value=".jsp" />  
</bean>  

<bean id="propertyConfigurer" 
    class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:*.properties</value>
        </list>
    </property>
</bean>

<bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">
    <property name="messageConverters">
          <list>
               <ref bean="mappingJacksonHttpMessageConverter" />
          </list>
    </property>
</bean>


<bean id="mappingJacksonHttpMessageConverter" 
    class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">  
    <property name = "supportedMediaTypes">  
        <list>  
            <bean class="org.springframework.http.MediaType">  
                <constructor-arg index="0" value="text"/>  
                <constructor-arg index="1" value="plain"/>  
                <constructor-arg index="2" value="UTF-8"/>  
            </bean>  
            <bean class="org.springframework.http.MediaType">  
                <constructor-arg index="0" value="*"/>  
                <constructor-arg index="1" value="*"/>  
                <constructor-arg index="2" value="UTF-8"/>  
            </bean>  
            <bean class="org.springframework.http.MediaType">  
                <constructor-arg index="0" value="text"/>  
                <constructor-arg index="1" value="*"/>  
                <constructor-arg index="2" value="UTF-8"/>  
            </bean>  
            <bean class="org.springframework.http.MediaType">  
                <constructor-arg index="0" value="application"/>  
                <constructor-arg index="1" value="json"/>  
                <constructor-arg index="2" value="UTF-8"/>  
            </bean>  
        </list>  
    </property>  
</bean>   
```
 
### 三、JSP页面

 这里做一个示范：当在一个输入框输入一个ISBN后，该输入框失去焦点时立刻查询出书籍的相关信息并显示在页面上而无需刷新整个页面

 
```
<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>添加书籍</title>
</head>
<body>
    <script type="text/javascript">
        var sysMsg = "${sysMsg}";
        if(sysMsg != "") alert(sysMsg);
    </script>

    <table>
        <td>ISBN：</td>
        <td><input type="text" name="isbn" id="bookIsbn"/></td>
    </table>
    <p>书名:<div id="bookMsg"></div></p>
    <p>持有量:<div id="bookHave"></div></p>

    <!--必须要先引入jQuery库-->
    <script type="text/javascript" src="/library/js/jquery-1.4.4.min.js"></script>
    <script>
        $(function(){
            $("#bookIsbn").blur(function(){
                $.ajax({
                    //发送请求URL，可使用相对路径也可使用绝对路径
                    url:"findBook.do",
                    //发送方式为GET，也可为POST，需要与后台对应
                    type:"GET",
                    //设置接收格式为JSON
                    dataType:"json",
                    //编码设置
                    contentType:"application/json;charset=utf-8",
                    //向后台发送的数据
                    data:{
                        isbn:$("#bookIsbn").val(),
                    },
                    //后台返回成功后处理数据，data为后台返回的json格式数据
                    success:function(data){
                        if(data != null){
                            $("#bookName").val(data.name);
                            $("#bookHave").val(data.haveNum);
                        }
                        else{
                            $("#bookName").val("该书籍不存在");
                        }
                    }
                    //查询错误处理
                    error:function(){
                        $("#bookName").val("查询异常");
                    }
                });
            });
        });

    </script>
</body>
</html>
```
 
### 四、控制器部分

 处理JSON的控制器方法需要使用@ResponseBody注解修饰，这样会自动将book实例中的变量以json格式返回   
 @RequestParam中的参数必须要与$.ajax()中data的数据对应

 
```
@ResponseBody
@RequestMapping("/findBook.do")
public Book checkBook(@RequestParam("isbn")String isbn)
{
    SqlSession sqlSession = MyBatisUtil.getSqlSessionFactory().openSession();
    Book book = null;
    try
    {
        BookMapper bookMapper = sqlSession.getMapper(BookMapper.class);
        book = bookMapper.selectBookByISBN(isbn);
        log.info("查询书籍成功");
    }
    catch(Exception e)
    {
        log.error("异常：", e);
    }
    finally
    {
        sqlSession.close();
        log.info("SQLsession已关闭");
    }
    return book;
}
```
 Book实体类

 
```
package com.library.entity;

public class Book implements java.io.Serializable 
{
    private static final long serialVersionUID = -8784440456544318126L;

    private Integer id;
    private String name;
    private String isbn;
    private Integer haveNum;
    //省略构造器及getter、setter方法
}

```
   
  