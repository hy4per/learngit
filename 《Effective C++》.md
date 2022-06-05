<h1> <center>《Effective C++》笔记</center></h1>

# **一、基础C++习惯**

## **1.视C++由四个次语言组成：**

- C
- Object-Oriented C++ (C with Classes)
- Template C++ (泛型编程部分)
- STL (容器，迭代器，适配器，空间分配器，仿函数，算法)

## **2.宁可以编译器替换预处理器：**

- 对于单纯常量，最好以const对象或enums代替#define
- 对于形似函数的宏，最好改用inline函数替换#define
- **enum hack** - 行为更像#define而不是const，如果不希望别人得到你的常量成员的指针或引用可以用enum hack代之。没有#define可视范围难以控制，不利于调试的缺点。和#define一样不会导致非必要的内存分配。

## **3.尽可能使用const：**

- 将某些东西声明为const可帮助编译器侦测出错误用法。const可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体
- 编译器强制实施bitwise constness
- 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复

## **4.确定对象被使用前已先被初始化**

- 为内置型对象进行手工初始化，因为C++不保证初始化它们
- 构造函数最好使用成员初值列，而不要在构造函数本体内使用赋值操作。初值列列出的成员变量，其排列次序应该和它们在class中的声明次序相同
- 为免除“跨编译单元之初始化次序”问题，请以local static对象替换non-local static（即在函数里面声明static对象）

# **二、构造/析构/赋值运算**

## **5.了解C++默默编写并调用哪些函数**

- 六大默认成员函数：构造函数、拷贝构造函数、析构函数、赋值运算符、取地址运算符、const取地址运算符。 当被调用时才会被编译器创建出来。

## **6.若不想使用编译器自动生成的函数，就应该明确拒绝**

```c++
//继承Uncopyable阻止对象被拷贝
class Uncopyable{
protected:
    Uncopyable() {}
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable&);//设置成私有防止copying
    Uncopyable& operator=(const Uncopyable&);
}
```

- 为驳回编译器自动提供的机能，可将相应的成员函数声明为private并且不予实现。
- 继承Uncopyable这样的基类也是一种做法。(例如Boost的noncopyable类)

## **7.为多态基类声明virtual析构函数**

- 带多态性质的基类应该声明一个virtual析构函数。如果class带有任何virtual函数，它就应该拥有一个virtual析构函数。
- class的设计目的如果不是作为基类使用，或不是为了具备多态性，就不该声明virtual析构函数

## **8.别让异常逃离析构函数**

- C++并不禁止析构函数吐出异常，但并不鼓励这样做
- 在两个异常同时存在的情况下，程序若不是结束执行就是导致不明确行为
- 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们（不传播）或结束程序。
- 如果客户需要对某个操作函数允许期间抛出的异常做出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作。

## **9.绝不在构造和析构过程中调用virtual函数**

- 在构造和析构期间不要调用virtual函数，因为这类调用从不下降至derived class（比起当前执行构造函数和析构函数的那层）

## **10.令operator=返回一个reference to \*this**

## **11.在operator=中处理“自我赋值”**

- 确保当对象自我赋值时operator=有良好行为。其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序（先用指针记住原来的data，新赋值后再删除原先的data）、copy-and-swap技术
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

## **12.复制对象时勿忘其每一个成分**

- Copying函数（复制构造函数、赋值操作符）应该确保复制“对象内的所有成员变量”及“所有base class成分”（1.复制所有local成员变量，2.调用所有base classes内的适当的copying函数）
- 不要尝试以某个copying函数实现另一个copying函数。应该将共同机能放进第三个函数中，并由两个copying函数共同调用

# **三、资源管理**

## 13.以对象管理资源

```c++
//一个工厂函数
//返回指针，指向Investment继承体系内的动态分配对象，调用者有责任删除它。
Investment* createInvestment();

void f()
{
    Investment * pInv = createInvestment();//调用factory函数
    //...
    delete pInv; //释放pInv所指对象
}
```

有些情况下f() 可能无法删除它得自createInvestment得投资对象，例如“...”区域内的一个过早的return语句；对createInvestment的使用位于某循环内，而该循环由于某个continue或goto语句过早退出；最后一种可能是“...”区域内的语句抛出异常。

delete被略过去，我们泄露的不只是内含投资对象的那块内存，还包括那些投资对象所保存的任何资源。

为了确保createInvestment返回的资源总是被释放，我们需要将资源放进对象内，当控制流离开f() ，该对象的析构函数会自动释放那些资源。

**以对象管理资源**的两个关键想法：

- **获得资源后立刻放进管理对象内**。createInvestment返回的资源当作只能指针的shared_ptr的初值。（**RAII**，资源取得时机便是初始化时机，在获得一笔资源后于同一语句内以它初始化某个管理对象）
- **管理对象运用析构函数确保资源被释放**。如果资源释放动作可能抛出异常，则比较棘手，参考条款8解决这个问题。

shared_ptr在其析构函数内做delete而不是delete[]操作，那意味在动态分配的数组身上使用shared_ptr是个馊主意。可通过一个可调用对象解决。

```c++
bool del(int *p)
{
    delete [] p;
}
//使用函数
share_ptr<int> ptr(new int[100], del);
//使用lambda表达式
shared_ptr<int> ptr(new int[100], [](int *p){delete [] p});
```

请记住：

- 为防止资源泄露，请使用RAII对象。它们在构造函数中获得资源并在析构函数中释放资源。
- 常用的RAII类比如shared_ptr。

## 14.在资源管理类中小心coping行为

假设使用C API函数处理类型为Mutex的互斥对象，共有lock和unlock两函数可用。

为了确保不会忘记将一个被锁住的Mutex解锁，建立一个class来进行管理。

```c++
class Lock{
    public:
    	explicit Lock(Mutex* pm)
            :mutexPtr(pm)
        { lock(mutexPtr);}
    
    	~Lock() { unlock(mutexPtr);}
    private:
    	Mutex *mutexPtr;
}
```

如果Lock对象被复制，将会发生什么？

```c++
Lock ml1(&m); //锁定m
Lock ml2(ml1); //将ml1复制到ml2身上
```

针对RAII对象可能被复制，大多数时候可能做的选择：

- **禁止复制**。许多时候允许RAII对象被复制并不合理。

- **对底层资源祭出“引用计数法”**。

  ```c++
  //有时候想要做的释放动作是解除而非删除，可以指定“删除器”
  //那是一个函数或函数对象，当引用次数为0时便被调用
  class Lock{
  public:
  	explicit Lock(Mutex* pm) //以某个Mutex初始化shared_ptr
       :mutexPtr(pm, unlock)//并以unlock为删除器
      {
           lock(mutexPtr.get());//条款15提到get
      }
  private:
      std::shared_ptr<Mutex> mutexPtr;
  };
  ```

- **复制底部资源**。深拷贝。
- **转移底部资源的拥有权**。如unique_ptr。

请记住，

- 复制RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。
- 普遍而常见的RAII类 copying行为是：禁止copying，引用计数法。

## 15.在资源管理类中提供对原始资源的访问
