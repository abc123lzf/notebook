---
title: 算法题中常用的C++ STL
date: 2017-12-19 11:48:37
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/78793680]( https://blog.csdn.net/abc123lzf/article/details/78793680)   
  ## 一、栈（stack）

 stack实现了一种先进后出的数据结构，使用时需要包含stack头文件

 
#### C++定义stack语法：

 
```
stack<int> s;//int为栈的数据类型，可以为string,double等
```
 
#### C++中stack的基本操作有：

 1、出栈：如 s.pop() 注意并不返回出栈的元素   
 2、进栈：如 s.push(x)   
 3、访问栈顶元素：如s.top();   
 4、判断栈空：如 s.empty() 栈为空时返回true   
 5、返回栈中元素个数，如：s.size()

 
#### 下面举一个简单的例子：

 
```
#include <iostream>
#include <stack>
using namespace std;

int main()
{
    std::stack<int> s;      //定义一个int类型的stack
    for(int i=0;i<10;i++)   //将数字0~9依次入栈
        s.push(i);

    std::cout << s.empty() << std::endl; //将输出0，表示栈不为空

    for(int i=0;i<10;i++)
    {
        std::cout << s.top() << std::endl;//输出栈顶的元素，依次为9、8、...、0
        s.pop();    //出栈
    }
    std::cout << s.empty() << std::endl;//将输出1，表示栈为空
    return 0;
}
```
 
#### 下面来看两个应用到栈的算法题：

 
##### 例题一：

 题目描述：   
 假设现要编制一个满足下列要求的程序：对一个表达式中只含有两种括号：圆括号和方括号，其嵌套的顺序随意，要求判断该表达式的括号是否匹配。   
 输入：   
 多组输入，每行输入一组括号组成的字符串。   
 输出：   
 对于每一组测试用例，如果匹配输出YES，否则输出NO   
 样例输入：   
 [()([])][]()   
 ([]())]   
 样例输出：   
 YES   
 NO

 
##### 思路：从该字符串第一个字符开始，遇到右括号时将该括号压入栈，遇到左括号时检查栈顶元素是否匹配（如’)’与’(‘匹配，’)’与’[‘不匹配），如果匹配，则弹出栈顶元素，如果不匹配则输出”NO”，结束该样例。最后还需要判断栈是否为空，如果为空，则输出”YES”,否则输出”NO”。

 
##### 答案：

 
```
#include <iostream>
#include <string>
#include <stack>
using namespace std;
int main()
{
    /*freopen("in.txt","r",stdin);
    freopen("out.txt","w",stdout);*/

    std::string str;
    std::stack<char> s;
    bool isMate;

    while(std::cin >> str)
    {
        isMate = true;
        for(int i = 0;i < str.size();i++)
        {
            if(str[i]=='('||str[i]=='[')
                s.push(str[i]);
            else if(str[i]==')')
            {
                if(!s.empty())
                {
                    if(s.top()!='(')
                    {
                        isMate = false;
                        std::cout << "NO" << std::endl;
                        break;
                    }
                    else
                        s.pop();
                }
                else
                {
                    isMate = false;
                    std::cout << "NO" << std::endl;
                    break;
                }
            }
            else if(str[i]==']')
            {
                if(!s.empty())
                {
                    if(s.top()!='[')
                    {
                        isMate = false;
                        std::cout << "NO" << std::endl;
                        break;
                    }
                    else
                        s.pop();
                }
                else
                {
                    isMate = false;
                    std::cout << "NO" << std::endl;
                    break;
                }
            }

        }
        if(isMate)
        {
            if(s.empty())
                std::cout << "YES" << std::endl;
            else
                std::cout << "NO" << std::endl;
        }

        while(!s.empty())
            s.pop();
    }
    return 0;
}
```
 
##### 例题二：

 中缀表达式转后缀表达式   
 时间限制：3000 ms | 内存限制：65535 KB   
 来源：[http://acm.nyist.net/JudgeOnline/problem.php?pid=467](http://acm.nyist.net/JudgeOnline/problem.php?pid=467)   
 描述   
 人们的日常习惯是把算术表达式写成中缀式，但对于机器来说更“习惯于”后缀式，关于算术表达式的中缀式和后缀式的论述一般的数据结构书都有相关内容可供参看，这里不再赘述，现在你的任务是将中缀式变为后缀式。   
 输入   
 第一行输入一个整数n，共有n组测试数据（n<10)。   
 每组测试数据只有一行，是一个长度不超过1000的字符串，表示这个运算式的中缀式，每个运算式都是以“=”结束。这个表达式里只包含+-*/与小括号这几种符号。其中小括号可以嵌套使用。数据保证输入的操作数中不会出现负数。   
 数据保证除数不会为0   
 输出   
 每组都输出该组中缀式相应的后缀式，要求相邻的操作数操作符用空格隔开。   
 样例输入   
 2   
 1.000+2/4=   
 ((1+2)*5+1)/4=   
 样例输出   
 1.000 2 4 / + =   
 1 2 + 5 * 1 + 4 / =

 
#### 思路：[http://blog.csdn.net/sgbfblog/article/details/8001651](http://blog.csdn.net/sgbfblog/article/details/8001651)

 答案：

 
```
#include<iostream>
#include<stack>
#include<string>
using namespace std;

int main()
{
    /*freopen("in.txt","r",stdin);
    freopen("out.txt","w",stdout);*/

    string str; //str为输入的中缀表达式
    string exp; //exp存储输出的后缀表达式
    stack<char> s;
    char ch;
    int t;
    cin >> t;
    while (t--)
    {
        cin >> str;
        str.insert(str.length(), "#");  //在中缀表达式加一个#字符作为结束标记
        for (int i = 0; i < str.length(); i++)
        {
            ch = str[i];
            if (ch >= '0'&&ch <= '9' || ch == '.')  //如果为数字则直接输出
            {
                if (str[i + 1] >= '0'&&str[i + 1] <= '9'||str[i + 1] == '.')//用来判断是不是个位数
                    exp += ch;
                else
                {
                    exp += ch;
                    exp += ' ';
                }
            }
            else if (ch == '+' || ch == '-')
            {
                while (!s.empty()&&s.top()!='(')    //弹出栈中元素直到遇到左括号或栈为空为止
                {
                    exp += s.top();     //取栈顶元素并存储在字符串exp中
                    exp += ' ';
                    s.pop();    //栈顶元素弹出
                }           
                s.push(ch);     //将当前操作符压入栈
            }
            else if (ch == '*' || ch == '/')
            {
                while (!s.empty() && s.top() != '+'&&s.top() != '-'&&s.top()!='(') //弹出栈中元素直到遇到加号、减号或者左括号为止
                {
                    exp += s.top(); //取栈顶元素并存储在字符串exp中
                    exp += ' ';
                    s.pop();    //栈顶元素弹出
                }
                s.push(ch);     //将当前操作符压入栈
            }
            else if (ch == '(') //遇到左括号直接压入栈但不输出
                s.push(ch);
            else if (ch == ')') //遇到右括号则弹出栈中元素直到遇到左括号为止
            {
                while (s.top() != '(')
                {
                    exp += s.top();
                    exp += ' ';
                    s.pop();
                }
                s.pop();   //弹出左括号
            }
            else if (ch == '#') //遇到结束符则弹出栈中所有元素
            {
                while (!s.empty())
                {
                    exp += s.top();
                    exp += ' ';
                    s.pop();
                }
            }
        }
        cout << exp << "=" << endl;
        exp = "";   //清空字符串
    }
    return 0;
}
```
 
## 二、动态数组（vector）

 C++中的vector是一个可以改变大小的数组，当解题时无法知道自己需要的数组规模有多大时可以用vector来达到最大节约空间的目的。使用时需要包含vector头文件。

 
#### 定义一个一维动态数组的语法为

 
```
vector<int> a;  //int为该动态数组的元素数据类型，可以为string、double等
```
 
#### 定义一个二维动态数组的语法为

 
```
vector<int*> a; //三维数据类型为int**，以此类推。
```
 
#### C++中vector的基本操作有：

 1、push_back(x) 在数组的最后添加元素x。   
 2、pop_back() 删除最后一个元素，无返回值。   
 3、at(i) 返回位置i的元素。   
 4、begin() 返回一个迭代器，指向第一个元素。   
 5、end() 返回一个迭代器，指向最后一个元素的下一个位置。   
 6、front() 返回数组头的引用。   
 7、capacity(x) 为vector分配空间   
 8、size() 返回数组大小   
 9、resize(x) 改变数组大小，如果x比之前分配的空间大，则自动填充默认值。   
 10、insert 插入元素   
 ①a.insert(a.begin(),10); 将10插入到a的起始位置前。   
 ②a.insert(a.begin(),3,10) 将10插入到数组位置的0-2处。   
 11、erase 删除元素   
 ①a.erase(a.begin()); 将起始位置的元素删除。   
 ②a.erase(a.begin(),begin()+2); 将0~2之间的元素删除。   
 12、rbegin() 返回一个逆序迭代器，它指向最后一个元素。   
 13、rend() 返回一个逆序迭代器，它指向的第一个元素前面的位置。   
 14、clear()清空所有元素。

 
#### 这里举一个简单的例子：

 
```
#include <iostream>
#include <algorithm>
#include <vector>
using namespace std;

int main()
{
    vector<int> a;
    vector<int> b;

    for (int i = 0; i < 10; i++)    //向数组a依次添加0~9
        a.push_back(i);

    a.swap(b);      //将数组a元素与数组b元素交换
    cout << a.size() << " " << b.size() << endl;    //此时应当输出 0 10

    for (vector<int>::iterator it = b.begin(); it != b.end(); it++)//从第一个元素开始遍历数组元素
        cout << *it << " ";     //依次输出0~9
    cout << endl;

    b.erase(b.begin() + 1);     //删除位置1的元素，即元素1.
    cout << b.size() << endl;   //由于删除了一个元素，此时输出应当为8

    for (vector<int>::reverse_iterator rit = b.rbegin(); rit != b.rend(); ++rit)//逆向输出数组元素
        cout << *rit << " ";    //应当输出9 8 7 6 5 4 3 2 0
    cout << endl;

    b.resize(9);    //将数组空间设定为9，相当于比之前多了1个位置
    b.push_back(20);//在尾部添加元素20

    for (vector<int>::iterator it = b.begin(); it != b.end(); it++)
        cout << *it << " ";  //应当输出0 2 3 4 5 6 7 8 9 20

    return 0;
}
```
 
#### 下面是运用到vector的例题：

 
##### 题目名称：Where is the Marble? (Uva 10474)

 链接：[https://vjudge.net/problem/UVA-10474](https://vjudge.net/problem/UVA-10474)   
 简要翻译：例题：   
 大理石在哪儿   
 现有N个大理石，每个大理石上写了一个非负整数、首先把各数从小到大排序   
 然后回答Q个问题。每个问题问是否有一个大理石写着某个整数x，如果是，还要   
 回答哪个大理石上写着x。排序后的大理石从左到右编号为1~N。(在样例中，为了   
 节约篇幅，所有大理石的数合并到一行，所有问题也合并到一行。)输入以0 0结束

 样例输入：   
 4 1   
 2 3 5 1   
 5   
 5 2   
 1 3 3 3 1   
 2 3   
 0 0   
 样例输出：   
 CASE# 1：   
 5 found at 4   
 CASE# 2：   
 2 not found   
 3 found at 3 

 
#### 思路：先将大理石编号从小到大排序，然后从第一个问题开始，遍历大理石，如果找到则输出该大理石的数组下标+1，如果没有找到则输出not found，接着从第二个问题开始，以此类推。

 答案：

 
```
#include <iostream>
#include <algorithm>
#include <vector>
using namespace std;

int main()
{
    /*freopen("in.txt","r",stdin);
    freopen("out.txt","w",stdout);*/

    vector<int> a;  //定义一个动态数组a，用来存储大理石编号
    vector<int> find;   //定义动态数组find，存储待查找的大理石编号
    int N, Q;
    int tmp, count = 0;
    bool flag;
    while (cin >> N >> Q)
    {
        if (N == 0 && Q == 0)
            break;
        for (int i = 0; i < N; i++)
        {
            cin >> tmp;
            a.push_back(tmp); //依次输入大理石编号
        }
        for (int i = 0; i < Q; i++)
        {
            cin >> tmp;   
            find.push_back(tmp);//依次输入待查找编号
        }

        sort(a.begin(), a.end());   //对数组a进行从大到小排序
        flag = false;               //标记是否找到
        cout << "CASE# " << ++count << ":" << endl;
        for (int i = 0; i < Q; i++) //从第一个问题开始
        {
            flag = false;
            for (int j = 0; j < N; j++)
                if (find.at(i) == a.at(j)) //如果找到
                {
                    cout << find.at(i) << " found at " << j + 1 << endl;
                    flag = true;
                    break;
                }
            if (!flag) //没有找到
                cout << find.at(i) << " not found" << endl;
        }
        a.clear();  //清空数组，为下一组数据准备
        find.clear();
    }
    return 0;
}
```
 
## 三、集合(set)

 C++中集合(set)类似于数学上的集合，即每个元素只能出现一次，使用该容器需要包含set头文件。

 
#### 定义一个set的语法为：

 
```
set<int> s; //int为集合的数据类型，可以为string,double等
```
 
#### C++中set的基本操作有：

 1、begin() 返回一个迭代器，指向第一个元素。   
 2、end() 返回一个迭代器，指向最后一个元素的下一个位置。   
 3、clear()清空set的所有元素。   
 4、empty() 判断是否为空。   
 5、size() 返回当前元素个数   
 6、erase(it) 删除迭代器指针it指向的元素。   
 7、insert(a) 插入元素a   
 8、count() 查找某个元素出现的次数，只有可能为0或1。   
 9、find() 查找某个元素出现的位置，如果找到则返回这个元素的迭代器，如果不存在，则返回s.end()

 
#### 这里举一个简单的例子：

 
```
#include <iostream>
#include <algorithm>
#include <set>
using namespace std;

int main()
{
    set<int> s;
    s.insert(20);
    s.insert(10);
    s.insert(30);
    s.insert(10);

    cout << s.size() << endl; //将输出3，因为集合中元素不能重复
    for (set<int>::iterator it = s.begin(); it != s.end(); it++)
        cout << *it << " ";     //将输出10 20 30，集合会自动排序
    cout << endl;

    //将输出1 0
    cout << count(s.begin(), s.end(), 20) << " " << count(s.begin(), s.end(), 40) << endl;

    s.erase(s.find(10));    //删除元素10
    for (set<int>::iterator it = s.begin(); it != s.end(); it++)
        cout << *it << " "; //将输出20 30
    return 0;
}
```
 
#### 下面是一些运用到set的例题：

 
##### 题目名称：Andy’s First Dictionary

 链接：[https://vjudge.net/problem/UVA-10815](https://vjudge.net/problem/UVA-10815)   
 题目简述：读入一篇英语文章，包含各种标点符号，要求按字典序输出文章中所有不同的单词。   
 样例输入：   
 Adventures in Disneyland   
 Two blondes were going to Disneyland when they came to a fork in the   
 road. The sign read: “Disneyland Left.”   
 So they went home.   
 样例输出：   
 a   
 adventures   
 blondes   
 came   
 disneyland   
 fork   
 going   
 home   
 in   
 left   
 read   
 road   
 sign   
 so   
 the   
 they   
 to   
 two   
 went   
 were   
 when   
 答案：

 
#### 思路：从该文章的第一个字符开始遍历，遇到大写字母则直接改为小写字母，遇到标点符号则以空格替代。然后通过字符串流存储在一个集合中，最后打印该集合的元素即为答案。

 
```
#include <iostream>
#include <algorithm>
#include <set>
#include <string>
#include <sstream>
using namespace std;

int main()
{
    set<string> s; //定义一个集合s，用来存储单词
    string str, tmp;
    while (cin >> str) //读入一个单词（可能包含标点符号）
    {
        for (string::iterator it = str.begin(); it != str.end(); it++) //遍历该单词元素
        {
            if (isalpha(*it))  //判断是否是英文字母
                *it = tolower(*it); //转换为小写
            else
                *it = ' ';  //如果是标点符号则用空格替代它
        }
        stringstream sstr(str);  //定义一个字符串流并将处理后的单词赋值给它
        while (sstr >> tmp)   //将该流输入到字符串tmp中，因为可能包含空格所以需要使用循环
            s.insert(tmp);    //插入到集合s中，若集合中已经存在该单词则不会插入
    }
    for (set<string>::iterator it = s.begin(); it != s.end(); it++)
        cout << *it << endl;
    return 0;
}
```
 
## 四、队列(queue)

 queue实现了一种先进先出的数据结构，使用时需要包含queue头文件。

 
#### 定义一个queue的语法为：

 
```
queue<int> q;   //int为队列的数据类型，可以为string,double等
```
 
#### C++中queue的基本操作有：

 1、入队，如：q.push(x) 将元素x置于队列的末端   
 2、出队，如: q.pop() 同样不会返回弹出元素的值   
 3、返回队首元素，如：q.front();   
 4、返回队尾元素，如：q.back();   
 5、判断是否为空，如：q.empty();   
 6、返回队列元素个数，如：q.size();

 
#### 这里举一个简单的例子:

 
```
#include <iostream>
#include <queue>
using namespace std;

int main()
{
    queue<int> q;
    for(int i = 0;i < 10;i++)   //将0~9依次入队
        q.push(i);

    cout << q.front() << " " << q.back() << endl; //这里应当输出0和9

    //依次输出0、1、...、9
    for(int i = 0;i < 10;i++)
    {
        cout << q.front() << " ";
        q.pop();
    }
    return 0;
}
```
 
## 总结：

 通过学习C++常用模板，在实际刷题过程中可以有效减少代码出错的几率和调试的难度，思路更清晰，代码更加简洁明了，可以有效提高刷题的速度和准确性。   
 如果该博客有错误的地方，欢迎指正。

   
  