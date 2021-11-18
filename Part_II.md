# Item5: 了解C++默默编写并调用哪些函数

C++编译器会主动为所有空类声明一个拷贝构造函数、拷贝赋值操作符和析构函数，甚至是default构造函数，所有这些函数都是public且inline的。

- 这仅是声明，直到函数被调用，才会真正去定义实现它们。
- 编译器实现的拷贝构造函数和拷贝赋值操作符只是单纯地将来源对象的每一个non-static成员对象拷贝到目标对象。

虽然大多数情况下这种默认的行为是能被接受的，但一旦涉及到类内引用、类内指针、const成员以及虚属性，问题将变得更加复杂。

```c++
template<typename T>
class NamedObject {
    NamedObject(std::string& name, const T& value);
    ...
private:
    std::string& nameValue;
    const T objectValue;
};

std::string newDog("Persephone");
std::string oldDog("Satch");

NamedObject<int> p(newDog, 2);
NamedObjecy<int> s(oldDog, 36);

p = s;
```

上述例子涉及两个点：类内引用和const成员。假如使用默认拷贝赋值操作符，那么p.nameValue所指的string对象将被修改，但C++并不允许”reference改指向不同的对象“，因此C++将拒绝编译这一行；对于p.objectValue的const成员也同理。

书中并没有提到类内指针，但涉及到指针，你就必须要考虑类内成员是否有深拷贝需求。一个简单的例子就是string类，它有很多种实现方式，但简单来说它就是一个char*对象，，假如你是用浅拷贝那么在使用中会引发一系列问题：

> 浅拷贝只是拷贝了指针，那么在对象块结束调用析构函数之时，由于要析构多个对象，但这些对象的指针成员由都指向同一块内存，导致重复delete；
>
> 浅拷贝使得多个对象的指针指向同一块内存，任何一方的变动都会影响到另一方；
>
> 极端情况下会引起内存泄漏(我感觉应该是直接core dump)。

当然深拷贝也并不是必须的，假如有了解过std::string源码，会发现标准库中采用引用计数的方式来避免浅拷贝的这些问题。

# Item6: 若不想使用编译器自动生成的函数，就该明确拒绝

在某些特殊场景，比如说单例模式，我们需要禁止拷贝行为，但编译器总是会默认帮你自动生成它们，书中给了两个解决方案：

- 将被禁止生产的函数声明为private并缺省实现，可以阻止类外调用；而由于缺省实现，那么类内调用会触发一个链接错误。
- 设计一个Uncopuable类，并私有继承它，那么无论谁想要进行copy行为，编译器都能意识到自己无法调用base class的copy函数，于是报错。

在C++11中可以直接使用`= delete`来显示禁止编译器自动生成某函数。

# Item7: 为多态基类声明virtual析构函数

大部分设计模式都是关于C++三大特性的使用经验，多态是C++区别于C的最重要的一点，考虑一个简单工厂：

```c++
class TimeKeeper {
public:
    TimeKeeper();
    /* virtual */ ~TimeKeeper();
    ...
};

class AtomicClock: publick TImeKeeper { ... };
class WaterClock: publick TImeKeeper { ... };
class WristClock: publick TImeKeeper { ... };

TimeKeeper* getTimeKerrper(...);
```

简单工厂根据传入的参数来决定new出来的derived class对象，并返回一个base class指针，返回的对象将位于heap上，这就意味着用户用户需要自行维护它的生存周期。当base class的析构函数不是虚函数，在调用delete删除对象时，仅仅会调用base class的析构函数，往往会导致对象的derived成分未被销毁。因此原则就是：polymorphic base classes应该声明一个virtual析构函数，防止指向子类的基类指针在被释放时至局部销毁了该对象。

如果class带有任何virtual函数，它就应该拥有一个virtual析构函数；但class并不具备polymorphic那它就不该声明virtual析构函数。这是因为一旦使用虚函数，编译器将为这个类生成一个虚函数表，并在运行时通过查该变来确定自身到底应该调用哪一个函数，这会在时间和空间上造成额外开销。

如果一个类型没有被设计成基类，又有被误继承的风险，请在类中声明为`final`（C++ 11），这样禁止派生可以防止误继承造成上述问题。

# Item8: 别让异常逃离析构函数

我必须承认从这里开始《Effective C++》就让我越看越抓狂，这一Item的主题是“不应该在析构函数中处理异常”，但作为一个养成了一定编程习惯的人来说，我是非常排斥使用异常处理的，因为它实在是太低效了并且debug的流程根本不会如预想的顺利，因此程序崩溃往往是一种比较好的bug，它比内存泄漏这种玄学问题要更佳直接。

因为我并不熟悉异常处理相关之时而解决方案也是我日常编程所用的，所以这一案例不做过多分析。其实我觉得对于接下来的几个案例，总结来说就是——保持良好的编程习惯，不要让一个“区块”承担过多的责任。在实际开发过程中，对于一个承担业务逻辑的类直觉告诉我除了构造/析构函数，我们还应该实现一系列Init/Create/Start/Pause/Stop/Destory等功能性函数，保持构造/析构函数的简单有利于程序的调试。

# Item9: 绝不在构造和析构过程中调用virtual函数

```c++
class Transaction {
   Transaction();
    virtual void logTransaction() const = 0;	// pure virtual function
    
    ...
};

Transaction::Transaction()
{
    ...
    logTransaction();
}

class BuyTransaction: public Transaction {
    virtual void logTransaction() const;
    ...
};

class SailTransaction: public Transaction {
    virtual void logTransaction() const;
    ...
};

BuyTransaction b;
```

不加验证合理性的，书中想通过上述代码阐述多态环境下构造函数的执行过程。C++基础告诉我们derived class的构造函数的调用顺序为先调用base class构造函数在调用derived class构造函数，因此BuyTransaction类的对象b在构造时会先调用Transaction类的构造函数，而Transaction类的构造函数调用了一个virtual函数。问题于是诞生——我们构造的是BuyTransaction类的对象，这在一定程度上意味着我们在调用virtual函数时，应该期望调用的是derived class的实现，可目前执行层级位于base class中，因此实际上被调用的是base class的实现，这可能不符合我们的期望。

- base class构造期间virtual函数绝不会下降到derived class层级，即这时的virtual函数不是virtual函数。一个直观的解释就是假如在base class构造期间(derived class部分尚未初始化)去调用derived class的函数将导致未定义的行为。
- 在多态环境下，构造函数和析构函数在执行过程中，**涉及到了对象类型从基类到子类，再从子类到基类的转变**，即在不同的运行时期会以对应的继承层级的身份存在。

书中这一条例的几个例子在我看来都有些矛盾：

- logTransaction被声明为pure virtual函数，这意味着Transaction为抽象类，对于抽象类的简单理解就是它无法实例化，于是这个类纯粹是为了提供接口的，这意味着对于纯虚函数的实现我们交由derived class定制，而原则上抽象类自身并不会实现纯虚函数，因此在抽象类中调用纯虚函数本身就是一种无法理解的行为。当然书中在最后也提到了这一点，改为基类调用impure virtual函数，那么情况就如条例所言：禁止调用。
- 针对书中提出的问题：如何确保没有一次Transaction继承体系上的对象被创建，就会有适当版本的log Transaction函数被调用，我的回答是每次创建对象都手动调用，这或许会有些直接但是它并不会造成损失。我认同书中的“令derived class将必要的构造信息向上传递至base class构造函数”，但这个说法似乎回避了这一问题，因为**必要的构造信息**不等于**所需要传递的信息**，书中的解决方法显然放弃了polymorphic，继承的一个优点在于拓展性，在继承状态下基类是无法去处理子类的额外信息的，因此通过基类去处理子类向上传递的信息也是一种不stable的行为，除非你能确保它的普适性。

# Item10: 令operator=返回一个reference to *this

简单来说：这样做可以让你的赋值操作符实现“连等”的效果：

```C++
x = y = z = 10;
```

在设计接口时一个重要的原则是，**让自己的接口和内置类型相同功能的接口尽可能相似**，所以如果没有特殊情况，就请让你的赋值操作符的返回类型为`ObjectClass&`类型并在代码中返回`*this`吧。

# Item11: 在operator=中处理“自我赋值”

“自我赋值”发生在对象被赋值给自己时，这是一种看似愚蠢但很容易被人轻视的操作，通常需要假借指针完成。

```c++
a[i] = a[j];	// i和j相等时，自我赋值
*px = *py;		// px和py指向同一对象，自我赋值
```

自我赋值在大多数时候不会引起大的问题，但假如这个对象管理了一定的资源(通常是指针)，那么情况就变得有所不同，开发人员可能会陷入“在停止使用资源之前意外释放了它”的陷阱中。这基于这样一个事实：对于赋值操作，我们通常会先清空 左值对象原本的内存或资源所有权，再完成赋值操作。

```c++
class Bitmap { ... };
class Widget {
    ...
private:
    Bitmap* pb;
};

Widget& Widget::operator=(const Widget& rhs)
{
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

如上例Widget的operator=的实现，如果发生自我赋值，那么将先把自己的pb成员所指向的对象释放掉，然后又把pb指向释放掉的资源，这明显会导致野指针。

解决的方法有很多，一个简单的操作就是在每次赋值之前先进行一个“证同测试”：

```c++
Widget& Widget::operator=(const Widget& rhs)
{
    if (this == &rhs) return *this;

    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

从我个人角度而言，这是一种简单且高效的做法，但书中认为它缺少安全性，是基于这样一个事实：假如`new Bitmap`发生异常，或者说是所有真正的赋值操作完成之前发生异常，返回的Widget对象都将是具有隐含问题的对象，因此书中提供了另外一种方法：通过拷贝和延迟删除来规避异常处理。

```c++
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap* pOrig = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrig;
    return *this;
}
```

另外一种方式是使用copy and swap技术：

```c++
class Bitmap { ... };
class Widget {
    void swap(Widget& rhs);
    ...
private:
    Bitmap* pb;
};

Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);
    sawp(temp);
    return *this;
}
```

# Item12: 复制对象时勿忘其每一个成分

“复制每一个成分”：

- 复制所有local成员变量：每当类中新增成员时，需要在copying函数中增加对应的处理
- 在类继承中，derive classes的copying函数应该调用所有base classes内的适当的copying函数

除此之外，拷贝构造函数和拷贝赋值操作符，他们两个中任意一个不要去调用另一个，这虽然看上去是一个避免代码重复好方法，但是是荒谬的。其根本原因在于拷贝构造函数在构造一个对象——这个对象在调用之前并不存在；而赋值操作符在改变一个对象——这个对象是已经构造好了的。因此前者调用后者是在给一个还未构造好的对象赋值；而后者调用前者就像是在构造一个已经存在了的对象。不要这么做！

