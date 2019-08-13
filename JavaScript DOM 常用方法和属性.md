---
title: JavaScript DOM 常用方法和属性
date: 2018-04-11 23:43:14
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/79903579]( https://blog.csdn.net/abc123lzf/article/details/79903579)   
  #### **1、getElementById方法**

 此方法返回一个有着给定id属性值的对象，例如一个img元素id值为logo：

 
```
<img src="a.jpg" id="logo" />
```
 那么我们可以通过该方法建立该属性值的对象并赋给变量img：

 
```
var img = document.getElementById('logo');
```
 注意：应该在body标签结束之前引入，不要放置于body标签开头。

 
#### **2、getElementsByTagName方法**

 这个方法会返回包含所有指定参数标签的对象数组，它的参数为标签的名字。   
 例如：

 
```
<h1>编程语言</h1>
<p>Java</p>
<p>C++</p>
<h1>Java WEB框架</h1>
<p>Spring MVC</p>
<p>Hibernate</p>
<script type="text/javascript">
    var p = document.getElementsByTagName('p');
    alert(p.length)
</script>
```
 将会输出4，因为该页面中有4个<p>元素

 
#### **3、getElementsByClassName方法**

 这个方法可以通过一个元素的class属性来访问元素，参数为类名，返回值为具有相同类名元素的数组。   
 例如：

 
```
<div class="test">
    <p>Java</p>
</div>
<div class="test">
    <p>C++</p>
</div>
<script type="text/javascript">
    var p = document.getElementsByClassName('test');
    alert(p.length)
</script>
```
 将会输出2

 
#### **4、getAttribute方法**

 此方法不属于document对象，它只能通过元素节点对象调用。   
 它的参数是你打算查询的属性的名字。   
 例如：

 
```
<p title="编程语言">Java、C++</p>
<p title="Java框架">Spring、Hibernate</p>
<script type="text/javascript">
    var paras = document.getElementsByTagName("p");
    for(var i = 0; i < paras.length; i++)
        alert(paras[i].getAttribute("title"));
</script>
```
 将会弹出两个提示框分别是：编程语言、Java框架

 
#### **5、setAttribute方法**

 setAttribute方法用来对属性节点的值做出修改。类似getAttribute，setAttribute也只能用于元素节点。它有两个参数：你需要修改的属性的名字和你想修改的值。

 
```
<p title="编程语言">Java、C++</p>
<p title="Java框架">Spring、Hibernate</p>
<script type="text/javascript">
    var paras = document.getElementsByTagName("p");
    for(var i = 0; i < paras.length; i++)
    {
        paras[i].setAttribute("title","hello");
        alert(paras[i].getAttribute("title"));
    }
</script>
```
 将会弹出两个提示框：都是hello

 
#### **6、children属性**

 同样children不属于document对象，它也只能通过元素节点对象调用   
 children属性可以用来获取一个元素的所有DOM元素，返回包含这个元素所有子元素的数组。   
 例如：

 
```
<p>Java、C++</p>
<p>Spring、Hibernate</p>
<script type="text/javascript">
    var para = document.getElementsByTagName("body")[0];
    alert(para.children.length)
</script>
```
 将会输出3，因为有两个<p…/>属性和一个<script…/>属性

 
#### **7、childNodes属性**

 这个方法类似children方法，不过childNodes方法会返回一个元素包含的所有元素，包含文本节点。   
 例如：

 
```
<p>Java、C++</p>
<p>Spring、Hibernate</p>
<script type="text/javascript">
    var para = document.getElementsByTagName("body")[0];
    alert(para.childNodes.length)
</script>
```
 将会输出6，因为有两个<p…/>属性和一个<script…/>属性，每个<p…/>属性和<script…/>各自包含一个文本元素。

 
#### **8、nodeType属性**

 nodeType属性值可以让我们知道自己正在与哪一种节点打交道，它只能通过元素节点对象调用。其中：   
 元素节点的nodeType为1，属性结点为2，文本节点为3

 
```
<p>Java、C++</p>
<p>Spring、Hibernate</p>
<script type="text/javascript">
    var para = document.getElementsByTagName("body")[0].childNodes;
    for(var i = 0; i < para.length; i++)
        alert(para[i].nodeType);
</script>
```
 将会输出3、1、3、1、3、1

   
  