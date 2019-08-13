---
title: Struts2+Hibernate 读取数据库存储有图片的Blob并将图片显示到前台页面
date: 2018-04-01 17:59:07
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79780079]( https://blog.csdn.net/abc123lzf/article/details/79780079)   
  ### 一、JSP页面部分

 获取数据库中的图片需要通过action来实现，具体如下

 
```
<img alt="头像" src="<s:url action='ShowUserProfilePicture'/>"/>
```
 可以看出，我们可以通过一个action返回一个URL来实现图片的展示。   
 如果需要参数，我们在action后设置一个s:param标签体来实现：

 
```
<img id="authorImg" src="<s:url action='Action'><s:param name='...' value='...'></s:param></s:url>" />
```
 
### 二、Hibernate部分

 这里为了方便描述，我们建立一个User类

 
```
package com.aplusBBS.nor;

import java.sql.Blob;
import java.sql.Timestamp;
import java.util.HashSet;
import java.util.Set;

import javax.persistence.Basic;
import javax.persistence.CascadeType;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.FetchType;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Lob;
import javax.persistence.OneToMany;
import javax.persistence.Table;

@Entity
@Table(name="user_inf")
public class User
{
    @Id @Column(name="user_id")
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Integer id;

    @Column(name="username")
    private String username;

    @Column(name="password")
    private String password;

    @Column(name="register_time")
    private Timestamp registerTime;

    //此处通过一个byte[]来存储图片，数据库也应当有一个Blob类型的列
    @Column(name="profile_picture")
    @Lob @Basic(fetch=FetchType.LAZY)
    private byte[] picture;

    public User() {}

    //省略一堆getter、setter方法
}

```
 
### 三、Action部分

 接下来我们创建一个上述JSP页面的图片展示action：ShowUserProfilePicture

 
```
public class ShowUserProfilePicture extends ActionSupport
{
    private static final long serialVersionUID = 1L;

    public String execute()
    {
        HttpServletRequest request = ServletActionContext.getRequest();
        HttpServletResponse response = ServletActionContext.getResponse();

        HttpSession session = request.getSession();

        User user = (User)session.getAttribute("user");

        //设置编码方式，用来解析图片（关键点一）
        response.setContentType("multipart/form-data");

        ServletOutputStream out = null;
        try 
        {
            //获取页面输出流（关键点二）
            out = response.getOutputStream();
            //获取User类中存储有图片二进制数据的数组（关键点三）
            byte[] img = user.getPicture();
            if(img == null)
            {
                return null;
            }
            else
                //将图片载入输出流（关键点四）
                out.write(user.getPicture());
            out.flush();
            out.close();
        }
        catch (IOException e) 
        {
            e.printStackTrace();
        }
        //必须返回null（关键点五）
        return null;
    }
}
```
 
### 四、struts.xml部分

 配置如下：

 
```
<action name="ShowAuthorImage" class="com.aplusBBS.action.bbs.ShowAuthorImage"></action>
```
   
  