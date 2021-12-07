本章节立足设计与声明，这是最能体现一个程序员功力的部分，遵守合理的设计规则能够让你开发出的程序更具竞争力。

# Item18: 让接口容易被正确使用，不易被误用

本条款旨在教你如何开发易用的接口，一个经验之谈就是——**不要假设用户具备任何先验知识，因为用户往往只关注他当前所需要的接口**。在日常的开发中，除非必要我也不会去穷尽第三方库实现上的细枝末节，在阅读源码之前我并不会知道这个接口实现上的细节，因此当我按照规则调用接口(编译通过)却并未获得预期结果时是很自闭的，所以一个优秀的设计应该尽可能的从**语法层面**并在**编译之时运行之前**，帮助接口的调用者规避可能的风险。

## 使用**外覆类型（wrapper）**提醒调用者传参错误检查，将参数的附加条件限制在**类型本身**

```c++
class Date {
public:
    Data(int month, int day, int year);
    ...
};

Date d(30, 3, 1995);	// 正确调用
Date d(2, 30, 1995);	// 错误调用
```

这或许是大多数人对于Date的设计情况，并且对于上述错误调用的情况，可以**在运行时**交由构造函数来判断，但这样的做法更像是一种责任转嫁——调用者只有在尝试过后才能发现错误。同时我们需要意识到我们对于运行时错误的处理手段总是有限的，这时候往往你不可避免的会导致程序终止。

因此我们可以在设计参数类型时将其抽象出来，书中的例子使用外覆类型来区别天数、月份和年份：

```c++
struct Day {
    explicit Day(int d): val(d) {}
    int val;
};

struct Month {
    explicit Month(int d): val(d) {}
    int val;
};

struct Year {
    explicit Year(int d): val(d) {}
    int val;
};

class Date {
public:
    Data(const Month& month, const Day& day, const Year& year);
    ...
};
```

但同时考虑到限制值的需求(日期仅12个月份等)，可以直接使用enum class(C++11引入的强枚举类型)。

```c++
enum class Month:int {
    January = 1, ......, December = 12  
};

class Date {
public:
    Data(const Month& month, const Day& day, const Year& year);
    ...
};
```

**注：**书中例子在最后提到了Item4中non-local static对象初始化的问题，在我看来似乎不是一个场景。

## 从**语法层面**限制调用者**不能做的事**

接口的调用者往往无意甚至没有意识到自己犯了个错误，所以接口的设计者必须在语法层面做出限制。一个比较常见的限制是加上`const`，比如在`operate*`的返回类型上加上`const`修饰，可以防止无意错误的赋值`if (a * b = c)`。

## 接口应表现出与内置类型的一致性

确保自定义类型的接口和内置类型的一致性，即保持各种类型之间具备兼容性，可以有效防止调用者犯错误。STL容器的接口十分一致，每个STL容器都有一个名为size的成员函数，能够告诉调用者目前容器内有多少个对象，但对于Java来说不同类型具有不同的size含义的method或property，这徒增了调用者的学习成本，并且容易导致误用。

## 从语法层面限制调用者**必须做的事**

这一点要求开发者尽可能从设计上强制要求调用者以一种长期稳定的方式调用接口：

```c++
Investment* createInvestment();
std::shared_ptr<Investment> createInvestment();
```

强制要求用户以智能指针对象的方式使用Investment对象，可以避免用户犯下资源泄漏的错误。

在书籍的最后对`std::shared_ptr`的特性做了一点补充，`std::shared_ptr`能够绑定自定义删除器，这能够解决一个“cross-DLL problem”：对象在一个DLL中被new创建，却在另一个DLL内被delete销毁。

这个问题的核心在于：**在多态环境下**，执行delete操作的DLL无法正确获取另一个DLL中关于这个对象的析构信息，但使用`std::shared_ptr`绑定自定义删除器使得对象在销毁时能够正确溯源到这个对象所在DLL中的析构信息。

# Item19: 设计class犹如设计type

本条款提醒我们设计class需要注意的细节，但并没有给每一个细节提出解决方案，只是提醒而已。每次设计class时最好在脑中过一遍以下问题：

- 对象该如何创建销毁：包括构造函数、析构函数以及new和delete操作符的重构需求。
- 对象的构造函数与赋值行为应有何区别：构造函数和赋值操作符的区别，重点在资源管理上。
- 对象被拷贝时应考虑的行为：pass by value时调用拷贝构造函数。
- 对象的合法值是什么？成员函数必须进行错误检查工作，最好在语法层面、至少在编译时应对用户做出监督。
- 新的类型是否应该复合某个继承体系，这就包含虚函数的覆盖问题。
- 新类型和已有类型之间的隐式转换问题，这意味着类型转换函数和非explicit函数之间的取舍。
- 新类型是否需要重载操作符。
- 什么样的接口应当暴露在外，而什么样的技术应当封装在内（public和private）
- 新类型的效率、资源获取归还、线程安全性和异常安全性如何保证。
- 这个类是否具备template的潜质，如果有的话，就应改为模板类。

# Item20: 宁以pass-by-reference-to-const替换pass-by-value

缺省情况下C++函数以by value方式传递或返回对象，除非另外指定，否则函数参数都是以实际实参的副本为初值，而调用端所获的的亦是函数返回值的一个副本，**这些副本由copy构造函数产出，使得pass-by-value成为开销昂贵的操作**。

而copy行为的引入，将导致一系列隐晦的问题：

- pass-by-value将导致一个临时变量的生成，表面上这包含了一次copy构造和一次析构，但实际上包含的行为是对象内所有成员变量的构造和析构，尤其涉及到深拷贝时开销将变得不容乐观。
- 对象切割问题：对于多态而言，将父类设计成按值传参，如果传入的是子类对象，仅会对子类对象的父类部分进行拷贝，即部分拷贝，而所有属于子类的特性将被丢弃，造成不可预知的错误，同时虚函数也不会被调用。

关于为什么需要const，一个直观的解释就是：pass-by-value其实隐含了一点就是**不会修改传入的对象**，因为函数体内使用的是副本而不是传入的对象本身。

当然面对内置类型和STL的迭代器与函数对象，我们通常还是会选择按值传参的方式设计接口。

最后书中讨论了一个特殊情况，在C++中reference的底层实现实际是指针，指针占用4/8字节(取决于OS)，因此某些足够小的类型，例如内置类型pass by value的效率会比pass by reference要高，**但这并不意味着小型类型就可以使用pass by value**。这部分细节我认为过于吹毛求疵了，我觉得还是使规范简单一点比较好，尽可能使用pass-by-reference-to-const即可。

# Item21: 必须返回对象时，别妄想返回其reference

这一条款阐述了函数返回对象的必要性——无论是返回stack或heap或静态区上对象的reference都或多或少会引发一系列的资源申请和回收的问题，多这一次拷贝也是没办法的事，最多就是指望着编译器来优化。

> 但是对于C++11以上的编译器，我们可以采用给类型编写“转移构造函数”以及使用`std::move()`函数更加优雅地消除由于拷贝造成的时间和空间的浪费。

**注：**截止文章更新的时间我还没掌握`std::move`的用法，因此在这无法展示一个实例，目前我的做法是传入一个引用实参，归根结底就是提前构造了一个对象，并没有节省拷贝，提高效率。

# Item22: 将成员变量声明为private

这一条款的核心在于将成员变量声明为private其所代表的**封装思想**。

书中提到的封装的好处主要有两个，与《C++ Prime》Chapter 7.2.1(Page 242)所阐述的一致：

- 被封装的类的具体实现细节可以随时改变，而无需调整用户级别的代码

将成员变量隐藏在函数接口的背后，可以为“所有可能的实现”提供弹性，当实现部分改变时，我们只需要检查类的代码本身以确认这次改变有什么影响，在不改变接口的情况下客户不需要更改类外的代码，最多只是重新编译类的实现本身。

- 确保用户代码不会无意间破坏封装对象的状态

将成员变量的访问权限设成private能够确保class的约束条件总是会获得维护，防止由于用户的原因造成的数据被破坏，因为只有实现部分的代码可能产生这样的错误。

此外所有的变量都是private了，那么所有的public和protected成员都是函数了，用户在使用的时候也就无需区分，这是语法一致性。

**注：**到此为止这一章节都是出于**用户是开发者的角度来考虑“接口的设计”**，但就**产品研发**来说，并没有太多具备二次开发能力的用户，用户往往期待的是现成的软件而不是SDK，这时实现的细节都将由你一个人来把握，因此个人认为“一致性”在这里并不是一个很重要的因素。而且必须要强调的一点就是在这种情况下，我们需要考虑投入产出比，权衡一个优秀的设计到底能带来多大的收益。这其实是一个很现实的问题——大部分小团队的交付压力是比较大的，花过多的时间在设计上也并不是领导者所期望的，而且小公司也确实不过于强调技术细节和迭代，为每一个成员变量设计Setter和Getter成员函数会让class变得异常臃肿，因此直接全局public不失为一个简单快捷的方法(当然这并不代表我赞成这种做法)。

# Item23: 宁以non-member、non-friend替换member函数

这一条款讨论了类的封装性和函数的包裹弹性，但需要注意的是这一条款仅对**不直接接触私有成员的函数生效**。

```c++
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
    ...
    void clearEverything();
};

void clearBrowser(WebBrowser& wb)
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```

上述代码分别实现了两个具有相同功能的member函数clearEverything和non-member函数clearBrowser，本条款告诉我们为何要用non-member的版本。

## 类的封装性

封装使我们能够改变事物而只影响有限客户，对成员封装性的一种量化标准是：愈多函数(member/friend)**可以**访问它，它的封装性就愈低。

clearEverything在设计上只是一系列member函数的组合，这样的函数隐含了一层意思：以一种不破坏原有成员封装性的方式实现功能。这样的想法很好，但clearEverything终究是还是一个member函数，它保留了对私有成员的访问权限，所以事实上它并不是一个稳定的实现手段，因为也许在后续的维护中忘记了这样的设计原则而破坏了封装性。于是non-member版本的clearBrowser是一种更为推荐的方式，因为它完全不可能增加“能够访问class内private成分”的函数数量。

## 包裹弹性和机能拓充性

将这类功能函数提取至类外部可以允许我们从**更多维度组织代码结构**，并**优化编译依赖关系**。

采用不同namespace来明确责任，将不同的高颗粒度函数和浏览器类纳入不同namespace和头文件，当我们使用不同功能时就可以include不同的头文件，而不用在面对cache的需求时不可避免的将cookies的工具函数包含进来，降低编译依存性。这也是`namespace`可以跨文件带来的好处。

```c++
// 头文件 webbrowser.h		针对class WebBrowserStuff自身
namespace WebBrowserStuff {
class WebBrowser { ... };		//核心机能
}

// 头文件 webbrowsercookies.h		针对WebBrowser和cookies相关的功能
namespace WebBrowserStuff {
	...							//与cookies相关的工具函数
}

// 头文件 webbrowsercache.h		针对WebBrowser和cache相关的功能、
namespace WebBrowserStuff {
	...							//与cache相关的工具函数
}
```

将所有功能函数放在多个头文件内但隶属同一个namespace，意味客户可以轻松拓展这一组函数，这是class定义式无法提供的性质。虽然客户可以继承class拓展机能，但derived calss无法访问base class的被封装的成员。

# Item24: 若所有参数皆需类型转换，请为此采用non-member函数

```c++
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
    // 重载'*'操作符
    const Rational operator* (const Rational& rhs) const;
private:
    ...
};

Rational oneEight(1, 8);
Rational oneHalf(1, 2);
Rational result = oneHalf * oneEight;
result = result * oneEight;

// 构造函数non-explicit，且参数具有默认值，允许发生int-to-rational的隐式转换
// 考虑以下两种混合调用形式
result = oneHalf * 2;
result = 2 * oneHalf;
```

先说结论，在操作符重载为成员函数的情况下，第一种调用是正确的，而第二种调用无法通过编译，书中的解释有些晦涩：

> 只有当参数被列于参数列(parameter list)内，这个参数才是隐式类型转换的合格参与者。地位相当于“被调用之成员函数所隶属的那个对象”——即this对象——的那个隐喻参数，绝不是隐式转换的合格参与者。这就是为什么上述第一次调用可以通过编译，第二次调用则否，因为第一次调用伴随一个放在参数列内的参数，第二次调用则否。

其实说白了就是由于Rational的构造函数是non- explicit的，因此对于第一种调用，编译器会自行完成隐式转换，也即作如下处理：

```c++
const Rational temp(2);
result = oneHalf * temp;
```

那么为什么第二种调用会无法通过编译呢，一个直白的解释就是根据操作符重载为成员函数的一个特性：**在操作符被重载为成员函数的情况下，操作符的第一个隐形参数this指针总是指向第一个操作数**，将调用展开为函数形式：

```c++
result = oneHalf.operator*(2);
result = 2.operator*(oneHalf);
```

通过上面的函数形式很明显就能看出，在操作符重载为成员函数的情况下编译器只能通过第一个操作数的类型来确定被调用的操作符到底属于哪一个类，因此第一个操作数(this指针所指向的隐喻参数)是不能进行隐式转换的，也即书中提到的”不是隐式转换的合格参与者“。

因此对于上例来说，整数2没有相应的class，也就没有相应的operator*函数，编译器会尝试寻找相关的non-member函数，假如没有定义的话编译器自然就会报错。

综上本条款的结论就是：**假如需要操作符的每一个参数(操作数)都允许发生隐式类型转换，就需要将操作符声明为non-member函数。**

**注：**书中一直强调“隐式转换”是一种不安全的行为，这一条款展示了作者认为唯一合适的地方，但是整数到有理数的例子实在是过于鸡肋，虽然除了操作符重载我也想不出应用场景。

# Item25: 考虑写出一个不抛异常的swap函数

出于个人私心，我并不想花精力在程序的异常处理上，因此这一条款在我看来更合适的名字应该是：写出一个全面且高效的swap函数。

`std::swap<T>(a, b);`是一个具有典型实现的模版函数，只要类型T支持copying，就能够帮助完成交换动作。

本条款考虑了各种场景下swap函数的不同实现：

- 当std::swap对你的实现效率不高时，提供一个swap member函数，并确定这个函数不抛出异常，同时提供一个non-member swap来调用member swap。
- 对于class，non-member swap最好是std::swap的特化；对于class template引入对应的namespace并实现non-member swap function template，以非显示调用的形式调用swap。

如条款31所说，一种有效的降低编译依赖的实现手法就是使用pImpl，也就是“以指针指向一个对象，内含真正数据”的对象，在这种情况下std::swap将变得异常低效，因为我们只需要置换指针，而std::swap却会实实在在的拷贝对象，因此我们可以考虑特化std::swap，使其对于自定义class只置换执行特定的置换操作。

```c++
class Widget {
public:
    ...
    void swap(Widget& other)
    {
        std::swap(pImpl, other.pImpl);
    }
};

namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        a.swap(b);	// 由于pImpl通常是私有成员，因此转而调用member函数来完成swap
    }
}
```

假设Widget和WidgetImpl都是模版类，那么情况将变得复杂起来。

```c++
template<typename T>
class WidgetImpl { ... };

template<typename T>
class Widget { ... };
```

成员函数的实现并不会增加增加内容，但对于std::swap的的处理，无论是偏特化还是重载都被禁止：

- C++禁止偏特化function template
- C++禁止重载std中的内容

解决的办法就是使用自定义namespace并在其中实现一个non-member swap函数，在调用时注意一下namespace，剩下的事情就交给C++的名称查找法则，编译器将自行选择最佳版本的swap进行调用。

```c++
namespace WidgetStuff {
    template<typename T>
    class WidgetImpl { ... };

    template<typename T>
    class Widget { ... };
    
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {
        a.swap(b);
    }
}

template<typename T>
void doSomething(T& obj1, T& obj2)
{
    using std::swap;
    ...
    swap(obj1, obj2);
    ...
}
```

















