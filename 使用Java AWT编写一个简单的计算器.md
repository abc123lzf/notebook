---
title: 使用Java AWT编写一个简单的计算器
date: 2017-11-26 15:48:40
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/78637002]( https://blog.csdn.net/abc123lzf/article/details/78637002)   
  **1、前言**   
 这个计算器是基于Java语言下的AWT图形库编写的。虽然功能简陋，但对于初学者而言可以为以后Java深入学习打下基础。   
 该计算器支持简单的四则运算。

 **2、演示与效果**   
 ![主界面](https://img-blog.csdn.net/20171126135244111?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 **3、各功能实现详解**   
 **（1）界面设置以及布局**   
 按钮分为两种类型，功能类型以及输入类型按钮。输入类型按钮为0~9数字键，其它则为功能按钮。   
 按键采用GridLayout布局管理器将其分为4*5大小格子。添加按键时需依次从左到右，从上到下添加到Panel类型容器中，再将Panel类型容器采用GridLayout布局管理器。   
 文本框采用BorderLayout布局管理器置于最上方。

 菜单栏中包含操作和其他两部分   
 操作中含有重置和退出，其他包含作者信息。   
 菜单栏需要用MenuBar类来实现，菜单栏中的元素需要用Menu实现，每个元素中按键采用MenuItem实现。   
 在创建MenuItem、Menu和MenuBar对象之后，调用Menu的add()方法将多个MenuItem组合成菜单（也可将另一个Menu对象组合起来形成二级菜单），再调用MenuBar的add()方法将多个Menu组合成菜单条，最后调用Frame对象的setMenuBar()方法来为该窗口添加菜单条。

 代码实现如下：

 
```
public class Calculator
{
    private Frame f = new Frame("计算器");
    private Frame info = new Frame("关于");
    private Button[] b = new Button[10];        //b为数字类型按钮
    private Button[] cal = new Button[8];       //cal为功能类型按钮    
    private TextField text = new TextField(30); //定义一个宽度为30的文本框

    //定义菜单栏
    private MenuBar mb = new MenuBar(); 
    private Menu option = new Menu("操作");     
    private Menu other = new Menu("其他");
    private MenuItem about = new MenuItem("关于");
    private MenuItem reset = new MenuItem("重置");
    private MenuItem exit = new MenuItem("退出");

    private Label information = new Label(); //用在“关于”窗口中

    //省略无关此部分代码

    public void init()
    {
        Panel p = new Panel();   //定义Panel类型容器p，用来盛装按钮

        for(int i = 0; i <= 9; i++)
            b[i] = new Button(""+i);  //将数字键依次从0开始命名
        //功能键命名
        cal[0] = new Button("+");
        cal[1] = new Button("-");
        cal[2] = new Button("*");
        cal[3] = new Button("/");
        cal[4] = new Button("=");
        cal[5] = new Button("退格");
        cal[6] = new Button(".");
        cal[7] = new Button("AC");
        //按顺序依次将按键添加到容器p中
        p.add(b[1]);p.add(b[2]);p.add(b[3]);p.add(cal[0]);
        p.add(b[4]);p.add(b[5]);p.add(b[6]);p.add(cal[1]);
        p.add(b[7]);p.add(b[8]);p.add(b[9]);p.add(cal[2]);
        p.add(cal[5]);p.add(b[0]);p.add(cal[6]);p.add(cal[3]);
        p.add(cal[7]);p.add(cal[4]);

        text.setEditable(false);               //将文本框设置为不可直接输入
        f.add(text,BorderLayout.NORTH);        //将文本框置于最顶端并添加至窗口f
        p.setLayout(new GridLayout(5,4,4,4));  //设置GridLayout布局管理器
        f.add(p);                              //将容器p添加到窗口f
        f.setMenuBar(mb);                      //设置菜单栏
        f.setSize(280,250)                     //将窗口大小设置为280*250像素
        //“关于”窗口
        info.setSize(250,100);
        info.add(information);
        information.setText("作者：APlus  CSDN博客:abc123lzf\n");
        info.setVisible(false);
        //省略无关此部分代码
    }
    //省略无关此部分代码
}
```
 **（2）事件监听器**   
 为了能使每个按钮能够处理用户的操作，必须要给各个组件加上事件处理机制。

 在事件处理中主要涉及到三类对象   
 事件源：事件发生的场所，通常为各个组件，例如按钮，窗口、菜单等。   
 事件：事件封装了GUI组件上发生的特定事情。如果程序需要获得GUI组件上所发生的事件的相关信息，都需要通过Event对象取得。   
 事件监听器：负责监听事件源所发生的事件，并对各种事件做出响应处理。

 现在回到计算器：   
 对于输入按键（0~9按键）而言，我们只需要在用户按下按键之后在文本框输入一个对应的数字即可。   
 代码实现：

 
```
public class Calculator
{
    //省略无关此部分代码
    class NumListener implements ActionListener
    {
        public void actionPerformed(ActionEvent e) 
        {
            if(e.getSource() == b[0])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"0");
                else
                {
                    text.setText("0");
                    flag2 = -1;
                }
            }
            //......直到b[9];
            else if(e.getSource() == b[9])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"9");
                else
                {
                    text.setText("9");
                    flag2 = -1;
                }
            }
        }       
    }
    public void init()
    {
        NumListener numListen = new NumListener();
        //将0~9按键添加一个事件监听器
        for(int i = 0; i <= 9; i++)
            b[i].addActionListener(numListen);
        //省略无关此部分代码
    }
}
```
 对于处理按键而言，我们需要在用户按下按键之后，对输入的数字进行计算，所以我们需要建立两个字符串型缓冲区。在第一次按下处理按键的时候，将文本框中信息复制到第一个缓冲区并将文本框清空，在第二次按下处理按键后，将文本框信息复制到第二个缓冲区并将两个缓冲区转化为double类型的值进行计算，并将计算结果显示在文本框中，再将计算结果复制到第一个缓冲区并清空第二个缓冲区，为下一次按下处理按键作准备。   
 代码实现：

 
```
public class Calculator
{
    //定义两个缓冲区
    private String load1 = new String();
    private String load2 = new String();

    private int flag = 0;          //用来标记是否第一次按下功能按键
    private int flag2 = -1;        //flag2标记按下第一个功能按键后，按下第一个数字按键时将文本框清空
    private int Key = -1;          //用来标记上一个功能按键，-1表示没有按下功能按键

    class CalListener implements ActionListener
    {
        public void actionPerformed(ActionEvent e) 
        {
            if(e.getSource() == cal[0])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 0;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) + Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                }
            }
            else if(e.getSource() == cal[1])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 1;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) - Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                }
            }
            else if(e.getSource() == cal[2])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 2;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) * Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                }
            }
            else if(e.getSource() == cal[3])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 3;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) /         Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                }
            }
            else if(e.getSource() == cal[4])
            {
                if(load1 != null && load2 == null)
                {
                    load2 = text.getText();
                    if(Key == 0)
                        text.setText(String.valueOf(Double.parseDouble(load1) + Double.parseDouble(load2)));
                    else if(Key == 1)
                        text.setText(String.valueOf(Double.parseDouble(load1) - Double.parseDouble(load2)));
                    else if(Key == 2)
                        text.setText(String.valueOf(Double.parseDouble(load1) * Double.parseDouble(load2)));
                    else if(Key == 3)
                        text.setText(String.valueOf(Double.parseDouble(load1) / Double.parseDouble(load2)));
                    Key = -1;
                    flag = 0;
                    flag2 = -1;
                    load1 = text.getText();
                    load2 = null;
                }
            }
        }
    }
    public void init()
    {
        CalListener calListen = new CalListener();
        load1 = load2 = null;

        cal[0].addActionListener(calListen);
        cal[1].addActionListener(calListen);
        cal[2].addActionListener(calListen);
        cal[3].addActionListener(calListen);
        cal[4].addActionListener(calListen);
        cal[5].addActionListener(e->               //退格按键
        {
            text.setText(text.getText().substring(0,text.getText().length()-1));
        });
        cal[6].addActionListener(e->               //小数点按键
        {
            text.setText(text.getText()+".");
        });
        cal[7].addActionListener(e->               //重置按键
        {
            load1 = load2 = null;
            Key = -1;
            flag2 = -1;
            flag = 0;
            text.setText("");
        });
    }
}
```
 对于AWT而言，需要用使用窗口监听器才能允许用户通过单击窗口右上角的“X”按钮来关闭窗口或是结束程序。我们可以创建一个类来继承WindowAdapter类并重写windowClosing()方法来实现。   
 代码实现：

 
```
class WindowListener extends WindowAdapter
{
    public void windowClosing(WindowEvent e)
    {
        System.exit(0);
    }
}
```
 对于菜单栏按键而言，类似于功能按键   
 代码实现：

 
```
public class Calculator
{
    public void init()
    {
        exit.addActionListener(e->System.exit(0));
        reset.addActionListener(e->
        {
            load1 = load2 = null;
            Key = -1;
            flag2 = -1;
            flag = 0;
            text.setText("");
        });

        about.addActionListener(e->
        {
            info.setVisible(true);
        });
    }
}
```
 **4、所有代码**

 
```
import java.awt.*;
import java.awt.event.*;

public class Calculator
{
    private Frame f = new Frame("计算器");
    private Frame info = new Frame("关于");

    private Button[] b = new Button[10];
    private Button[] cal = new Button[8];

    private MenuBar mb = new MenuBar();
    private Menu option = new Menu("操作");
    private Menu other = new Menu("其他");
    private MenuItem about = new MenuItem("关于");
    private MenuItem reset = new MenuItem("重置");
    private MenuItem exit = new MenuItem("退出");
    private Label information = new Label();

    private TextField text = new TextField(30);
    private int flag = 0;
    private int flag2 = -1;
    private int Key = -1;

    String load1 = new String();
    String load2 = new String();

    public void init()
    {
        Panel p = new Panel();
        NumListener numListen = new NumListener();
        CalListener calListen = new CalListener();

        load1 = load2 = null;

        for(int i = 0; i <= 9; i++)
            b[i] = new Button(""+i);

        info.setSize(250,100);
        information.setText("作者：APlus  CSDN博客:abc123lzf\n");

        for(int i = 0; i <= 9; i++)
            b[i].addActionListener(numListen);

        cal[0] = new Button("+");
        cal[1] = new Button("-");
        cal[2] = new Button("*");
        cal[3] = new Button("/");
        cal[4] = new Button("=");
        cal[5] = new Button("退格");
        cal[6] = new Button(".");
        cal[7] = new Button("AC");
        cal[0].addActionListener(calListen);
        cal[1].addActionListener(calListen);
        cal[2].addActionListener(calListen);
        cal[3].addActionListener(calListen);
        cal[4].addActionListener(calListen);
        cal[5].addActionListener(e->
        {
            text.setText(text.getText().substring(0,text.getText().length()-1));
        });
        cal[6].addActionListener(e->
        {
            text.setText(text.getText()+".");
        });
        p.add(b[1]);p.add(b[2]);p.add(b[3]);p.add(cal[0]);
        p.add(b[4]);p.add(b[5]);p.add(b[6]);p.add(cal[1]);
        p.add(b[7]);p.add(b[8]);p.add(b[9]);p.add(cal[2]);
        p.add(cal[5]);p.add(b[0]);p.add(cal[6]);p.add(cal[3]);
        p.add(cal[7]);p.add(cal[4]);

        text.setEditable(false);
        exit.addActionListener(e->System.exit(0));
        reset.addActionListener(e->
        {
            load1 = load2 = null;
            Key = -1;
            flag2 = -1;
            flag = 0;
            text.setText("");
        });
        cal[7].addActionListener(e->
        {
            load1 = load2 = null;
            Key = -1;
            flag2 = -1;
            flag = 0;
            text.setText("");
        });
        about.addActionListener(e->
        {
            info.setVisible(true);
        });

        option.add(reset);
        option.add(exit);
        other.add(about);
        mb.add(option);
        mb.add(other);
        f.setMenuBar(mb);


        f.add(text,BorderLayout.NORTH);
        p.setLayout(new GridLayout(5,4,4,4));

        info.add(information);
        info.addWindowListener(new WindowListener());
        f.addWindowListener(new WindowListener());
        f.add(p);
        f.setSize(280,250);
        f.setVisible(true);
    }

    class WindowListener extends WindowAdapter
    {
        public void windowClosing(WindowEvent e)
        {
            if(e.getSource() == f)
                System.exit(0);
            else if(e.getSource() == info)
                info.setVisible(false);
        }
    }

    class NumListener implements ActionListener
    {
        public void actionPerformed(ActionEvent e) 
        {
            if(e.getSource() == b[0])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"0");
                else
                {
                    text.setText("0");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[1])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"1");
                else
                {
                    text.setText("1");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[2])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"2");
                else
                {
                    text.setText("2");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[3])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"3");
                else
                {
                    text.setText("3");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[4])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"4");
                else
                {
                    text.setText("4");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[5])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"5");
                else
                {
                    text.setText("5");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[6])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"6");
                else
                {
                    text.setText("6");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[7])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"7");
                else
                {
                    text.setText("7");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[8])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"8");
                else
                {
                    text.setText("8");
                    flag2 = -1;
                }
            }
            else if(e.getSource() == b[9])
            {
                if(flag2 == -1)
                    text.setText(text.getText()+"9");
                else
                {
                    text.setText("9");
                    flag2 = -1;
                }
            }
        }       
    }

    class CalListener implements ActionListener
    {
        public void actionPerformed(ActionEvent e) 
        {
            if(e.getSource() == cal[0])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 0;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) + Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                    //flag = 0;
                }
            }
            else if(e.getSource() == cal[1])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 1;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) - Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                    //flag = 0;
                }
            }
            else if(e.getSource() == cal[2])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 2;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) * Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                    //flag = 0;
                }
            }
            else if(e.getSource() == cal[3])
            {
                if(flag == 0)
                {
                    load1 = text.getText();
                    flag = 1;
                    flag2 = Key = 3;
                }
                else if(flag == 1)
                {
                    load2 = text.getText();
                    text.setText(String.valueOf(Double.parseDouble(load1) / Double.parseDouble(load2)));
                    load1 = text.getText();
                    load2 = null;
                    //flag = 0;
                }
            }
            else if(e.getSource() == cal[4])
            {
                if(load1 != null && load2 == null)
                {
                    load2 = text.getText();
                    if(Key == 0)
                        text.setText(String.valueOf(Double.parseDouble(load1) + Double.parseDouble(load2)));
                    else if(Key == 1)
                        text.setText(String.valueOf(Double.parseDouble(load1) - Double.parseDouble(load2)));
                    else if(Key == 2)
                        text.setText(String.valueOf(Double.parseDouble(load1) * Double.parseDouble(load2)));
                    else if(Key == 3)
                        text.setText(String.valueOf(Double.parseDouble(load1) / Double.parseDouble(load2)));
                    Key = -1;
                    flag = 0;
                    flag2 = -1;
                    load1 = text.getText();
                    load2 = null;
                }
            }
        }
    }

    public static void main(String[] args)
    {
        new Calculator().init();
    }
}
```
   
  