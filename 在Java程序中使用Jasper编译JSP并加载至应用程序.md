---
title: 在Java程序中使用Jasper编译JSP并加载至应用程序
date: 2018-07-30 11:20:21
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/81253511]( https://blog.csdn.net/abc123lzf/article/details/81253511)   
  ### 一、前言

 Jasper是Tomcat自带的JSP编译器，它可以将JSP文件转换成Java源文件和class文件供Tomcat加载并使用。网上关于Jasper编译器如何使用的文章少之又少，在我翻阅了Tomcat官方文档后领悟到了使用方法。如果你想尝试自己写一个像Tomcat那样的Servlet容器，那么这篇文章适合你。

 先附上官方文档：[https://tomcat.apache.org/tomcat-7.0-doc/api/org/apache/jasper/JspC.html](https://tomcat.apache.org/tomcat-7.0-doc/api/org/apache/jasper/JspC.html)

 
### 二、准备

 要使用Jasper编译器，需要以下Jar包资源：   
 Ant   
 下载地址：[http://ant.apache.org/bindownload.cgi](http://ant.apache.org/bindownload.cgi)   
 Tomcat lib目录下的所有jar包

 
### 三、编译和加载

 **编译过程**

 
```
import org.apache.jasper.JspC;
public final class Test {
    public static void main(String[] args) {
        JspC jspc = new JspC(); //创建一个Jasper编译器对象
        jspc.setUriRoot("C:\\Tomcat\\webapps\\test"); //WEB应用存放路径
        jspc.setOutputDir("C:\\Tomcat\\work"); //JSP编译结果输出路径
        jspc.setCompile(true); //是否将JSP转换后的.java源代码文件转换成.class文件
        jspc.execute(); //确认提交
    }
}
```
 运行上述代码，可以发现C:\Tomcat\work目录下新建了两层文件夹：org/apache，在apache目录下保存了转换后的.java源代码文件和编译后的.class文件。因为JspC编译器默认包名格式为org.apache.{Web文件夹层次}。   
 例如：你的jsp文件放在了/jsp/index.jsp下，那么编译后的完整类名为org.apache.jsp.index_jsp。JSP类名为JSP文件名（不包括后缀名）加上”_jsp”。如果你的JSP文件放在了/WEB-INF/view目录下，那么编译后的JSP class类名会变成org.apache.WEB_002dINF.view.index_jsp，JSP编译器会默认将WEB-INF转换成WEB_002dINF。

 可能有朋友会问了为什么会将WEB-INF转换成WEB_002dINF。   
 其实不止是WEB-INF文件夹会这样，Jasper为了保证包名不包含非法的字符，文件夹名带有其它非英文、数字字符的（除了‘.’字符，该字符会被替换成’_’），都会将那个字符替换成unicode十六进制表示形式，如果文件夹名或JSP文件名以数字开头，会在前面加一个下划线字符。比如WEB-INF文件夹中的特殊字符为’-‘，’-‘字符在unicode字符集中对应0x002D，Jasper就会将那个’-‘替换成”_002d”。如果文件夹本身带有’_’字符，也同样会将其转换成”_005f”。   
 下面是具体的转换逻辑：

 
```
public static final String DEFAULT_JSP_PACKAGE = "org.apache";

/**
* 将JSP的URI转换成该JSP对应的类名
* @param uri JSP的URI路径,类似${webappName}/WEB-INF/view/index.jsp
* @return 该JSP的类名
*/
private static String parseJspURIToClass(String uri) {
    String[] split = uri.split("/");

    for(int j = 0; j < split.length; j++) {
        char[] str = split[j].toCharArray();
        for(int i = 0; i < str.length; i++) {
            char c = str[i];
            if(Character.isAlphabetic(c) || Character.isDigit(c) || c == '.') {
                continue;
            }
            split[j] = split[j].replace("" + c, String.format("_%04x", Character.codePointAt(new char[] {c}, 0)));
        }
        split[j] = split[j].replace(".", "_");
        if(Character.isDigit(str[0])) {
            split[j] = "_" + split[j];
        }
    }

    String result = DEFAULT_JSP_PACKAGE + '.';

    for(int i = 0; i < split.length; i++) {
        if(i == split.length - 1)
            result += split[i];
        else
            result += (split[i] + '.');
    }
    return result;
}
```
 如果你想自定义包名，可以在提交编译前加上。

 
```
jspc.setPackage("com.test");
```
 那么编译后的JSP包名就从org.apache变成了com.test。

 值得注意的是，如果你的JSP文件引用了标签库，例如JSTL标签库。那么你需要将JSTL的jar包放入WEB-INF下的lib文件夹中就可以了，Jasper编译器会自动检测lib目录下的标签库文件并加载。前提是你的UriRoot路径必须是你的Web应用根目录。

 如果你想只编译一个JSP文件，在提交前加上

 
```
jspc.setJspFiles("index.jsp");
```
 即可。

 **JSP类加载**   
 要想加载JSP class文件，我们需要自定义一个JSP类加载器并将父类加载器设置为该Web应用的类加载器。

 
```
public class JspClassLoader extends ClassLoader {

    private static final StringManager sm = StringManager.getManager(JspClassLoader.class);
    private static final Log log = LogFactory.getLog(JspClassLoader.class);

    private final String jspWorkPath;
    //类名(包括包名)和class文件二进制数据映射表
    private final Map<String, byte[]> map = new ConcurrentHashMap<>(24);
    //判断该类加载是否读入了class文件
    private boolean isLoad = false;

    /**
     * @param parent 父类加载器，设置为该Web应用的父类加载器
     * @param jspWorkPath 编译好的jsp class文件存放路径
     */
    public JspClassLoader(ClassLoader parent, File jspWorkPath) {

        super(parent);

        if(!jspWorkPath.exists())
            jspWorkPath.mkdir();

        this.jspWorkPath = jspWorkPath.getAbsolutePath();
    }

    @Override
    public Class<?> findClass(String name) throws ClassNotFoundException {
        //判断有没有将class文件加载至内存，如果没有则启动加载过程
        if(!isLoad) {
            synchronized(this) {
                if(!isLoad) {
                    startRead(new File(jspWorkPath));
                    isLoad = true;
                }
            }
        }

        byte[] classBytes = map.get(name);

        if(classBytes == null) {
            throw new ClassNotFoundException(sm.getString("JspClassLoader.findClass.e0", name));
        } else {
            map.remove(name);
            return defineClass(name, classBytes, 0, classBytes.length);
        }
    }
    /**
     * 将所有编译好的JSP class文件读入内存
     * @param file JSP class文件工作路径，在本服务器中，路径为/work/${HostName}/${contextName}
     */
    private void startRead(File file) {
        if(file.exists()) {
            File[] files = file.listFiles();

            if(files.length == 0) {
                return;
            } else {
                for(File file2 : files) {
                    if(file2.isDirectory()) {
                        startRead(file2);
                    } else {
                        if(file2.getName().endsWith(".class")) {
                            //将路径转换成标准的类名
                            String klass = file2.getAbsolutePath().replace(jspWorkPath + File.separator, "")
                                    .replace(".class", "").replace(File.separator, ".");

                            if(this.findLoadedClass(klass) != null)
                                continue;

                            byte[] b = loadJspClassFile(file2);
                            if(b == null)
                                continue;
                            //将加载好的class文件放入映射表
                            map.put(klass, b);
                        }
                    }
                }
            }
        }

    }

    /**
     * 读取编译好的JSP类文件并转换为字节数组
     * @param file 该JSP类文件路径
     * @return 该JSP类文件的字节数组
     */
    private byte[] loadJspClassFile(File file) {

        try {
            @SuppressWarnings("resource")
            FileInputStream fis = new FileInputStream(file);

            byte[] b = new byte[(int) file.length()];
            fis.read(b);
            return b;

        } catch (FileNotFoundException e) {
            log.error(sm.getString("JspClassLoader.loadJspClassFile.e0", file.getAbsolutePath()), e);
        } catch (IOException e) {
            log.error(sm.getString("JspClassLoader.loadJspClassFile.e1", file.getAbsolutePath()), e);
        }

        return null;
    }
}
```
 当我们想获取某个JSP类时，调用这个类加载器的loadClass方法并传入类名就可以了   
 有关类加载器基础的文章网上有很多，这里就不再赘述了。

   
  