---
title: Struts2+Hibernate 实现图片上传至数据库
date: 2018-04-04 23:07:02
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79561818]( https://blog.csdn.net/abc123lzf/article/details/79561818)   
  ### JSP页面部分

 为了实现上传图片，我们需要将上传图片部分设置为一个表单   
 具体代码如下：

 
```
<s:form action="UploadPicture" enctype="multipart/form-data">
        <s:file name="image" label="选择照片"/>
        <s:submit value="上传"/>
</s:form>
```
 <s:file/> 为该表单的元素，用来实现文件上传。   
 需要注意的是：我们需要将表单编码格式(即enctype属性)设置为multipart/form-data，用来表示该表单上传的数据为二进制数据。

 
### struts.xml部分

 为了实现识别该文件是否是图片和控制图片的大小的功能，我们需要为该action定义一个拦截器：fileUpload拦截器，实现上述功能需要给这个拦截器设置两个参数：maximumSize和allowedTypes，前者用来限制文件的大小，后者用来识别该文件的格式。参数maximumSize的值的单位为字节，比如1MB则写成1048576，allowedType的值为图片格式，用逗号分开。

 
```
<action name="UploadPicture" class="com.demo.UploadProfilePicture">
        <interceptor-ref name="defaultStack" />
        <interceptor-ref name="fileUpload" >
            <param name="maximumSize">1048576</param>
            <param name="allowedTypes">
                image/bmp,image/jpg,image/png,image/gif,image/jpeg
            </param>
        </interceptor-ref>
        <!--下面正常配置result,此处省略-->
</action>
```
 
### Action部分

 看下面的代码，其中标明了几个关键的地方：   
 HibernateUtil工具类参考：[https://blog.csdn.net/vipmao/article/details/51340525](https://blog.csdn.net/vipmao/article/details/51340525)   
 要求数据库存储图片对应的列的数据类型为Blob类型

 
```
public class UploadPicture extends ActionSupport
{
    private static final long serialVersionUID = 1L;

    //该表单的元素，类型为File，需要设置setter、getter方法（关键点一）
    private File image;

    public String execute()
    {
        HttpServletRequest request = ServletActionContext.getRequest();
        HttpSession session = request.getSession();

        Session hibernateSession = HibernateUtil.currentSession();
        Transaction tx = hibernateSession.beginTransaction();

        try 
        {
            //将该图片转为二进制输入流（关键点二）
            FileInputStream fis = new FileInputStream(getImage());
            //设置一个byte数组，数组长度为该图片的字节数目（关键点三）
            byte[] content = new byte[fis.available()];
            //二进制流输入到数组（关键点四）
            fis.read(content);
            //备注：User类是一个映射数据库表的类
            User user = (User)session.getAttribute("user");
            //设置该图片数组到这个对象的属性，该属性类型同样是byte[]类型（关键点四）
            user.setPicture(content);
            //更新数据库
            hibernateSession.update(user);
            tx.commit();
            HibernateUtil.closeSession();
            request.setAttribute("sysMsg", "上传成功");

            return SUCCESS;
        }
        //如果上传的文件不符合要求会被拦截器拦截，并抛出NullPointerException异常
        catch(NullPointerException e)
        {
            tx.rollback();
            request.setAttribute("sysMsg", "上传失败：文件过大或图片格式不符合要求");
            HibernateUtil.closeSession();
            return ERROR;
        }
        catch (Exception e)
        {
            e.printStackTrace();
            tx.rollback();
            HibernateUtil.closeSession();
            request.setAttribute("sysMsg", "服务器异常");
            return ERROR;
        }
    }

    public File getImage() {
        return image;
    }

    public void setImage(File image) {
        this.image = image;
    }
}

```
 如果想了解如何读取数据库中的图片到前台页面，可以参考我的这篇博客：   
 [https://blog.csdn.net/abc123lzf/article/details/79780079](https://blog.csdn.net/abc123lzf/article/details/79780079)

   
  