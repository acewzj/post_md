---
title: static
tags:
  - static
categories:
  - CPP
date: 2020-11-11 20:54:38
---

这篇文章记述了关于static关键字的知识点。

<!--more-->

## static面向过程

1、**静态全局变量**

在全局变量前，加上关键字static，该变量就被定义成为一个静态全局变量。

**静态全局变量有以下特点：**
• 该变量在全局数据区分配内存；
• 未经初始化的静态全局变量会被程序自动初始化为0（自动变量的值是随机的，除非它被显式初始化）；
• 静态全局变量在声明它的整个文件都是可见的，而在文件之外是不可见的；　

2、**静态局部变量**

在局部变量前，加上关键字static，该变量就被定义成为一个静态局部变量。

静态局部变量有以下特点：
• 该变量在全局数据区分配内存；
• 静态局部变量在程序执行到该对象的声明处时被首次初始化，即以后的函数调用不再进行初始化；
• 静态局部变量一般在声明处初始化，如果没有显式初始化，会被程序自动初始化为0；
• 它始终驻留在全局数据区，直到程序运行结束。但其作用域为局部作用域，当定义它的函数或语句块结束时，其作用域随之结束；

3、**静态函数**

在函数的返回类型前加上static关键字,函数即被定义为静态函数。静态函数与普通函数不同，它只能在声明它的文件当中可见，不能被其它文件使用。

定义静态函数的好处：
• 静态函数不能被其它文件所用；
• 其它文件中可以定义相同名字的函数，不会发生冲突；

## static面向对象

**4、静态数据成员**
在类内数据成员的声明前加上关键字static，该数据成员就是类内的静态数据成员。

可以看出，静态数据成员有以下特点：
• 对于非静态数据成员，每个类对象都有自己的拷贝。而静态数据成员被当作是类的成员。无论这个类的对象被定义了多少个，静态数据成员在程序中也只有一份拷贝，由该类型的所有对象共享访问。也就是说，静态数据成员是该类的所有对象所共有的。对该类的多个对象来说，静态数据成员只分配一次内存，供所有对象共用。所以，静态数据成员的值对每个对象都是一样的，它的值可以更新；

5、**静态成员函数**
　　与静态数据成员一样，我们也可以创建一个静态成员函数，它为类的全部服务而不是为某一个类的具体对象服务。静态成员函数与静态数据成员一样，都是类的内部实现，属于类定义的一部分。**普通的成员函数一般都隐含了一个this指针，this指针指向类的对象本身，因为普通成员函数总是具体的属于某个类的具体对象的。**通常情况下，this是缺省的。如函数fn()实际上是this->fn()。**但是与普通函数相比，静态成员函数由于不是与任何的对象相联系，因此它不具有this指针。从这个意义上讲，它无法访问属于类对象的非静态数据成员，也无法访问非静态成员函数，它只能调用其余的静态成员函数。**（上次被问到能不能在静态成员函数里调用非静态成员函数，答案在这里！！！！）



## 类中的静态变量

由于声明为static的变量只被初始化一次，因为它们在单独的静态存储中分配了空间，因此类中的静态变量**由对象共享。**对于不同的对象，不能有相同静态变量的多个副本。也是因为这个原因，**静态变量不能使用构造函数初始化。而是必须在类外进行初始化，不然编译能通过，链接却通不过。（2020.11补充）**

```c
// variables inside a class 

#include<iostream> 
using namespace std; 

class Apple 
{ 
    public: 
        static int i; 

        Apple() 
        { 
            // Do nothing 
        }; 
}; 

int main() 
{ 
    Apple obj1; 
    Apple obj2; 
    obj1.i =2; 
    obj2.i = 3; 

    // prints value of i 
    cout << obj1.i<<" "<<obj2.i; 
} 
--------------------
/tmp/cc6TTR48.o: In function `main':
/home/acewzj/CPlusPlusThings/basic_content/static/static_error_variable.cpp:21: undefined reference to `Apple::i'
/home/acewzj/CPlusPlusThings/basic_content/static/static_error_variable.cpp:22: undefined reference to `Apple::i'
/home/acewzj/CPlusPlusThings/basic_content/static/static_error_variable.cpp:25: undefined reference to `Apple::i'
/home/acewzj/CPlusPlusThings/basic_content/static/static_error_variable.cpp:25: undefined reference to `Apple::i'
collect2: error: ld returned 1 exit status
终端进程已终止，退出代码: 1  
```

您可以在上面的程序中看到我们已经尝试为多个对象创建静态变量i的多个副本。但这并没有发生。因此，类中的静态变量应由用户使用类外的类名和范围解析运算符显式初始化，如下所示：

```c
#include<iostream> 
using namespace std; 

class Apple 
{ 
public: 
    static int i; 

    Apple() 
    { 
        // Do nothing 
    }; 
}; 

int Apple::i = 1; 

int main() 
{ 
    Apple obj; 
    // prints value of i 
    cout << obj.i; 
} 
---------------
    1
```



## 类对象为静态

就像变量一样，对象也在声明为static时具有范围，直到程序的生命周期。

考虑以下程序，其中对象是非静态的。

```c
#include<iostream> 
using namespace std; 

class Apple 
{ 
    int i; 
    public: 
        Apple() 
        { 
            i = 0; 
            cout << "Inside Constructor\n"; 
        } 
        ~Apple() 
        { 
            cout << "Inside Destructor\n"; 
        } 
}; 

int main() 
{ 
    int x = 0; 
    if (x==0) 
    { 
        Apple obj; 
    } 
    cout << "End of main\n"; 
} 
---------------
Inside Constructor
Inside Destructor
End of main
```

在上面的程序中，对象在if块内声明为非静态。因此，变量的范围仅在if块内。因此，当创建对象时，将调用构造函数，并且在if块的控制权越过析构函数的同时调用，因为对象的范围仅在声明它的if块内。 如果我们将对象声明为静态，现在让我们看看输出的变化。

```c
#include<iostream> 
using namespace std; 

class Apple 
{ 
    int i; 
    public: 
        Apple() 
        { 
            i = 0; 
            cout << "Inside Constructor\n"; 
        } 
        ~Apple() 
        { 
            cout << "Inside Destructor\n"; 
        } 
}; 

int main() 
{ 
    int x = 0; 
    if (x==0) 
    { 
        static Apple obj; 
    } 
    cout << "End of main\n"; 
} 
--------------
Inside Constructor
End of main
Inside Destructor
```

您可以清楚地看到输出的变化。现在，在main结束后调用析构函数。这是因为静态对象的范围是贯穿程序的生命周期。

## 类中的静态函数

就像类中的静态数据成员或静态变量一样，静态成员函数也不依赖于类的对象。我们被允许使用对象和'.'来调用静态成员函数。但建议使用类名和范围解析运算符调用静态成员。

允许静态成员函数仅访问静态数据成员或其他静态成员函数，它们无法访问类的非静态数据成员或成员函数。为啥？

```c
#include<iostream> 
using namespace std; 

class Apple 
{ 
    public: 
        // static member function 
        static void printMsg() 
        {
            cout<<"Welcome to Apple!"; 
        }
}; 

// main function 
int main() 
{ 
    // invoking a static member function 
    Apple::printMsg(); 
} 
------------------------
Welcome to Apple!
```

## 阿里：如何在C++中定义一个不能被继承的类呢？

**想法一：私有构造函数来解决**
我们都知道在C++中子类的构造函数会自动调用父类的构造函数，子类的析构函数也会自动调用父类的析构函数。如果想要一个类无法被继承，那仫这个类的构造函数和析构函数必须是私有的，但是这样又会出现新的问题？就是这个类的构造函数和析构函数是私有的，我们如何获取和删除这个类的实例化对象呢？共有的静态函数就可以解决这个问题。

```c
class Base{
public:
    Base* getNewBase(){
        return new Base();
    }
    void deleteBase(Base* b){
       delete b;
    }
private:
    //私有化构造和析构函数
    Base();
    ~Base();
}

int main()
{
    //静态成员函数没有隐含this指针参数，使用类型::作用域访问符直接调用静态成员函数
    Base *b=Base::GetSpace();
    Base::DeleteSpace(b);
    system("pause");
    return 0;
}
```

这样的实现会不会存在什仫问题呢？通过观察上面的代码我发现实例化一个对象只能使用new来实例，也就是说这个新创建的对象只能在堆上生成，这就存在一定的局限性。那仫如何解决呢？
**想法二：利用虚拟继承的方法**
同样是将父类的构造函数私有化，但是却定义了一个模板类型的友元，因为友元的特性：如果一个类是另一个类的友元，友元类的每个成员函数都是另一个类的友元函数，都可访问另一个类中的保护或私有数据成员。我们在子类中调用父类的构造和析构函数是不会出错的。
这种想法比方法一的想法好的一点就是：既可以在堆上实例化对象也可以在栈上实例化对象。

```c
template<typename T>
class Base
{
    friend T;  //定义友元，子类可以访问父类的私有成员对象
private:
    Base()
    {
        cout<<"Base()"<<endl;
    }
    ~Base()
    {
        cout<<"~Base()"<<endl;
    }
};
//虚继承
class Sub:virtual public Base<Sub>  
{
public:
    Sub()
    {
        cout<<"Sub()"<<endl;
    }
    ~Sub()
    {
        cout<<"~Sub()"<<endl;
    }
};

int main()
{
    Sub s1;   //可以在栈上开辟
    Sub *s2=new Sub; //可以在堆上开辟
    delete s2;
    system("pause");
    return 0;
}
```

```c
执行结果--
    先执行 Sub s1 --> 执行Base的构造函数（因为可以访问友元的私有变量和函数） 
    输出：Base()  Sub()
    再执行 Sub *s2 = new Sub;-->在堆上创建对象
    输出：Base()  Sub()
    再执行delete s2;-->删除堆上创建的对象（删除时先执行子类的析构函数，再执行基类的析构函数）
    输出：~Sub() ~Base()
    最后停在pause这里，当按下任意按键时，栈上的对象也执行析构函数
    输出：按任意键继续 按下以后输出 ~Sub() ~Base() （这句话几乎看不到）
```



# static_cast

当编译器隐式执行类型转换时，大多数的编译器都会给出一个警告：

```
e:\vs 2010 projects\static_cast\static_cast\static_cast.cpp(11): warning C4244: “初始化”: 从“double”转换到“int”，可能丢失数据
```

　　使用static_cast可以明确告诉编译器，这种损失精度的转换是在知情的情况下进行的，也可以让阅读程序的其他程序员明确你转换的目的而不是由于疏忽。