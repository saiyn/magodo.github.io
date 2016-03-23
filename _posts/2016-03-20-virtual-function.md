---
layout: "post"
title: "虚函数(virtual function)"
categories:
- "cpp"
---

<!--more-->

***
Table of Content

* TOC
{:toc}
***

# 1. 子类对象指向基类的pointer和reference

子类的对象在实例化之后会包含“自身定义的部分”和“基类定义的部分”两部分。对于一个已有的对象，可以定义基类的指针或基类的引用。这些指针和引用只能访问“基类的部分”。

## 1.1 例子

{%highlight CPP linenos%}
#include <iostream>

class Animal
{
    protected:
        std::string m_name;

        /* 这里构造函数使用protected是为了防止别人直接创建Animal对象 */
        Animal(std::string name)
            :m_name(name)
        {}
    public:
        std::string Speak()
        {
            return "???";
        }
        std::string GetName()
        {
            return m_name;
        }
};

class Cat: public Animal
{
    public:
        Cat(std::string name)
            :Animal(name)
        {}

        std::string GetName()
        {
            return m_name;
        }

        std::string Speak()
        {
            return "Meow";
        }
};

class Dog: public Animal
{
    public:
        Dog(std::string name)
            :Animal(name)
        {}

        std::string GetName()
        {
            return m_name;
        }

        std::string Speak()
        {
            return "Woung";
        }
};


int main()
{
    Cat cat("Fred");
    std::cout << "I'm " << cat.GetName() << ": " << cat.Speak() << '\n';

    /* 基类的指针 */
    Animal *pAnimal = &cat;
    /* 基类的引用 */
    Animal &rAnimal = cat;
    std::cout << "I'm " << rAnimal.GetName() << ": " << rAnimal.Speak() << '\n';
    std::cout << "I'm " << pAnimal->GetName() << ": " << pAnimal->Speak() << '\n';
}
{%endhighlight%}

上例输出：

{%highlight CPP linenos%}
➜  pointer_reference_to_base ./a.out       
I'm Fred: Meow
I'm Fred: ???
I'm Fred: ???
{%endhighlight%}

从上例中可见，C++允许通过子类的对象定义其任意基类的指针和引用。同时，从输出中可以看到，基类的指针和引用调用的`Speak`函数是基类中定义的。

## 1.2 指向基类的指针和引用的应用

**情景1** 

假设有以下函数：

    void Report(Cat &cat)
    {
        std::cout << "I'm " << cat.GetName() << ": " << cat.Speak() << '\n';
    }

    void Report(Dog &dog)
    {
        std::cout << "I'm " << dog.GetName() << ": " << dog.Speak() << '\n';
    }

假象有30种不同动物，那么你就得为每种动物都定义一个`Report`函数。更好做法是，在他们共有的基类中定义这个Report函数，并且传入的参数类型为`Animal`。可是会遇到一个问题就是基类中的`Speak`定义的是"???".解决这个问题需要用到虚函数。

**情景2**

假设有以下数组：

    Cat acCats[] = { Cat("Fred"), Cat("Tyson"), Cat("Zeke") };
    Dog acDogs[] = { Dog("Garbo"), Dog("Pooky"), Dog("Truffle") };
     
    for (int iii=0; iii < 3; iii++)
        cout << acCats[iii].GetName() << " says " << acCats[iii].Speak() << endl;
         
        for (int iii=0; iii < 3; iii++)
            cout << acDogs[iii].GetName() << " says " << acDogs[iii].Speak() << endl;

由于数组中的元素必须是同类型的。因此，如果有30中动物，就要定义30组数组。更好的做法是，定义一个`Animal`的数组，传入的元素可以是`Cat`的对象，也可以是`Dog`的对象。可是，同样会遇到上面的问题。

# 2. 虚函数和多态（polymorphism）

虚函数是一种特殊的函数类型，在resolve某个特定签名的函数调用时会取当前object范围内的most-derived的版本。这种特性也被称为**多态**.

虚函数的语法是在函数声明的地方，在最前面加上`virtual`修饰符（注意：不要将虚函数的`virtual`和虚拟基类的搞混）。

## 2.1 基本用法

以上一节animal的问题为例：

{%highlight CPP linenos%}

#include <iostream>

class Animal
{
    protected:
        std::string m_name;

        /* 这里构造函数使用protected是为了防止别人直接创建Animal对象 */
        Animal(std::string name)
            :m_name(name)
        {}
    public:
        virtual std::string Speak()
        {
            return "???";
        }
        std::string GetName()
        {
            return m_name;
        }
};

class Cat: public Animal
{
    public:
        Cat(std::string name)
            :Animal(name)
        {}

        virtual std::string Speak()
        {
            return "Meow";
        }
};

class Dog: public Animal
{
    public:
        Dog(std::string name)
            :Animal(name)
        {}

        virtual std::string Speak()
        {
            return "Woung";
        }
};


void report(Animal &animal)
{
    std::cout << animal.GetName() << " says " << animal.Speak() << '\n';
}

int main()
{
    Cat cat("FeiMao");
    Dog dog("WangCai");

    report(cat);
    report(dog);
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
virtual_function > ./a.out                                                                                                                                                                        
FeiMao says Meow
WangCai says Woung
{%endhighlight%}

上面的改动是，分别在`Animal`,`Dog`,`Cat`三个类的`Speak`函数前面加上了`virtual`修饰符。

发生的过程是：传入`Cat`的对象到`report`函数，作为`Animal`（基类）的引用。这个引用在正常情况下只能访问`Cat`对象中`Animal`的部分。因此，在调用`Speak`函数的时候，会首先resolve为`Animal`中的版本。但是，由于加入了`virtual`修饰符，C++编译器会在这个引用所指向的对象的全部范围内（包括`Cat`部分 ）寻找most-derived的`Speak`函数，并最终调用该版本。

另一个例子：

{%highlight CPP linenos%}
#include <iostream>
using namespace std;

class A
{
    public:
        virtual void Say()
        {
            cout << "A\n";
        }
};

class B: public A
{
    public:
        virtual void Say()
        {
            cout << "B\n";
        }
};

class C: public B
{
    public:
        virtual void Say()
        {
            cout << "C\n";
        }
};

class D: public C
{
    public:
        virtual void Say()
        {
            cout << "D\n";
        }
};

int main()
{
    C obj;
    A &a_obj = obj;
    /* 这里输出的是"C",原因是a_obj拥有obj的数据（虽然在正常情况下只能访问A的部分），也即从A类到C类的数据。
       因此，对于虚函数的resolve也仅限于此范围内。 而不会resolve到D中的版本。
    */
    a_obj.Say();
}
{%endhighlight%}

输出： `C`

## 2.2 虚函数签名

CPP编译器只会对具有相同函数签名的虚函数尝试resolve到most-derived的版本。这其中包括了函数的参数的类型和数量，函数的返回值（大部分情况下）。

其中，对于函数的返回值的特例是，当基类虚函数的返回的是一个类的指针或者引用，那么在子类中重新定义的函数的返回值可以是该子类的指针或者引用。

## 2.3 virtual 修饰符

理论上，`virtual`修饰符只需要在most-base的类的相应函数加上即可。例如：

{%highlight CPP linenos%}

#include <iostream>
using namespace std;

class A
{
    public:
        virtual void Say()
        {
            cout << "A\n";
        }
};

class B: public A
{
    public:
        void Say()
        {
            cout << "B\n";
        }
};

class C: public B
{
    public:
        void Say()
        {
            cout << "C\n";
        }
};

class D: public C
{
    public:
        void Say()
        {
            cout << "D\n";
        }
};

int main()
{
    C c_obj;
    B &b_obj = c_obj;
    b_obj.Say();
}
{%endhighlight%}

输出： `C`

上例中，虽然在B和C的`Show`函数前都没有加`virtual`，但由于`A`前面是加过的，因此，C++编译器知道符合这个函数签名的，定义`A`的子类中的该函数都是虚函数。

在子类中加和不加`virtual`在功能上是一样的，不过建议加上。这样可以时刻提醒你这个函数是一个虚函数。

# 3. 虚拟析构函数，强制使用基类的虚拟函数 (virtual desctructor, override virtual)

## 3.1 虚拟析构函数(virtual desctructor)

如果你的类会被继承，那么你总是应该将这个类的析构函数定义成虚函数。

请看下例：

{%highlight CPP linenos%}

#include <iostream>
using namespace std;

class Base
{
    public:
        ~Base()
        {
            cout << "Base destructor\n";
        }
};

class Derived: public Base
{
    private:
        int *m_array;

    public:
        Derived(int length)
        {
            m_array = new int[length];
        }

        ~Derived()
        {
            cout << "Derived destructor\n";
            delete[] m_array;
        }
};

int main()
{
    Derived *p_derived = new Derived(5);
    Base *p_base = p_derived;
    delete p_base;
}
{%endhighlight%}

输出： `Base destructor`

显然，我们希望在`delete` `p_base` 的时候，是调用`Derived`的析构函数。因此，应该在两个类的析构函数前都加上`virtual`:


{%highlight CPP linenos%}

#include <iostream>
using namespace std;

class Base
{
    public:
        virtual ~Base()
        {
            cout << "Base destructor\n";
        }
};

class Derived: public Base
{
    private:
        int *m_array;

    public:
        Derived(int length)
        {
            m_array = new int[length];
        }

        virtual ~Derived()
        {
            cout << "Derived destructor\n";
            delete[] m_array;
        }
};

int main()
{
    Derived *p_derived = new Derived(5);
    Base *p_base = p_derived;
    delete p_base;
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
virtual_destruct_assignment_override > ./a.out 
Derived destructor
Base destructor
{%endhighlight%}

## 3.2 强制使用基类的虚函数(override virtual)

有些情况下需要使用某个虚函数在基类中的版本，只要直接使用`::`符号来显示地指定基类和相应函数即可。

# 4. 虚函数表(virtual table)

## 4.1 函数的 Early Binding 与 Late Binding

所谓的*Early Binding*在这里是指，在编译时可以确定函数的虚拟内存地址的情况。一般就是直接调用函数的情况。

所谓的*Late Binding*在这里是指，在编译时无法确定函数的实际地址。比如说有一个指向某个函数原型的函数指针，在运行时可能被赋予不同的函数的地址，因此只有在运行时才能确定。

## 4.2 虚函数表

虚函数的实现借助于**虚函数表**。虚函数表实质上使用的是上面提到的Late Binding。

首先，每一个使用虚函数（或者继承自某个定义了虚函数的类）的类都有它自己的一个虚函数表。这个表仅仅是一个由编译器在编译时创建的静态的数组。一个虚函数表为每一个虚函数保留一个表项，保存的是指向此类能够得到的该函数most-derived版本的指针。

其次，编译器还会为most-base的类加入一个public的隐藏指针，我们称之为`*__vptr`. `*__vptr`是在类的实例创建的时候被自动创建的(如果这个类是个子类，那这个指针继承自基类)，指向该类的虚函数表。不同于`*this`指针（只是编译器用来解析自引用的函数参数），`*__vptr`是一个真实的指针。

举个例子：

{%highlight CPP linenos%}
ass Base
{
public:
    FunctionPointer *__vptr;    // 仅仅是模仿编译器的动作
    virtual void function1() {};
    virtual void function2() {};
};
 
class D1: public Base
{
public:
    virtual void function1() {};
};
 
class D2: public Base
{
public:
    virtual void function2() {};
};
{%endhighlight%}

由于这里有三个类，因此编译器会创建3个虚函数表。

同时，这里为了模仿编译器为most-base的类创建的隐藏指针，在上面的例子中定义了该指针。因此，当Base创建一个对象时，它会有一个指向Base的虚函数表的指针；当D1或者D2创建一个对象时，它们也会有一个指向各自虚函数表的指针。

这个例子的三个virtual table如下图所示(右边部分)：

![vtable](/images/cpp/vtable.png)

这时候，对于下面的代码：

{%highlight CPP linenos%}
int main()
{
    D1 cClass;
    Base *pClass = &cClass;
    pClass->function1();
}
{%endhighlight%}

首先，*pClass*是指向Base的指针，因此它只指向i*cClass*中Base的部分。同时，由于`*__vptr`是Base的一部分，因此，*pClass*可以访问的到。然而，因为*cClass*中的`*__vptr`虽然继承自基类，但是其值实际指向D1的虚函数表。因此，最终*pClass*是指向D1的虚函数表，因此调用的是D1虚函数表中的function1，也即D1::function1.


