# Item 1: 视C++为一个语言联邦

C++的设计伊始仅是在C的基础上添加一部分面向对象的特性，但随着语言发展，当下的C++更像一个语言集合，其元素(sublanguage)共包含四个：

- C：C++是C的较高级解法，pointer/intelligent pointer，templates/overloading
- Object- Oriented：C with classes，类/封装/继承/多态/动态绑定
- Template：generic programming，通常情况下并不会过多涉及这部分内容
- STL：template programming的一个最佳实践，非常建议学习侯捷老师的STL相关课程

很有意思的是C++ Prime中也将内容划分为这四部分，不同的地方在于C++ Prime更像一本工具书，读者应该按需阅读而不是死磕语言。

- 对于内置类型而言，**pass-by-value**更高效
- 对于OO，由于user-defined constructor和deconstructor的存在，**pass-by- reference-to-const**会更好

# Item 2: 尽量以const，enum，inline替换#define

- “宁可以编译器替换与处理器”——尽可能减小预处理器引发错误的可能性

const和#define两个关键字是C++面试八股的热门，两者的区别网上有很多全面的分析：

- #define在预处理阶段替换展开，const具有类型检查，并且会加载进记号表中，于是在报错时会得到更为明晰的错误提示
- #define在重复展开的情况下会导致目标码出现多份数据拷贝，因此使用const常量相较而言会产生更小的目标码并且直到调用才会进行内存分配
- 类中的常量：#define并无作用域的概念，可以理解为#define全局生效，而const可以被限制在class内，但仅用const修饰的常量的生命周期与对象有关，对整个类而言依然是可变的，因此假如想要确保常量至多只有一份那么还需要加上static关键字。

书中的例子：

```c++
class GamePlayer {
private:
    static const int NumTurns = 5;		// 声明式而非定义式
    int scores[NumTurns]
    ...
};
```

需要注意的是在声明式中赋初值的操作仅局限于int类型，正确的作法应该是在GamePlayer类的实现文件中显式定义：`const int GamePlayer::NumTurns = 5;`。

接下来的特殊情况个人认为有些过于苛刻了，此外以这样的例子引入enum其实有些牵强，我觉得承认enum的局限性也未尝不可，毕竟假如有阅读过Linux kernel源码，可以发现无论是kernel还是driver，都会大量使用#define(非宏)而非const/enum，保持源码的优美和规范是一部分原因，但具体原因还在思考中。

## template inline function

```c++
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

int a = 5, b = 0;
CALL_WITH_MAX(++a, b);			// a累加2次
CALL_WITH_MAX(++a, b + 10)；	   // a累加1次
```

- 使用宏需要注意的一点就是对于参数需要增加括号以避免传递不正确的参数(值)。
- 另类的调用导致一些未知或者说不明显的错误，上述例子在两种调用情况下a累加的次数并不相同。当然这个的例子相对还是有点牵强，毕竟这种代码出现的概率还是非常小，而且在大型工程中为了跨文件/模块定义和传递对象和函数，宏定义或者extern是值得保留的。

不过宏的大量使用会增加代码的阅读难度，往往你为了找到一个正确的函数调用就得花费大量的时间，这也是我不喜欢阅读使用C语言大型库源码的原因。

因此我也更建议使用template inline函数：

```c++
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a : b);
}
```

它的高效分为两部分：

- template
- inline

细节暂时不展开，未来再补充。

# Item 3: 尽可能使用const

const指定一个语义约束——不该被改动的对象。

关于const关键字的语法细节建议阅读《C++ Prime》的Chapter 2.4(Page 53)，比较难以理解的是const修饰指针的情况，《C++ Prime》建议你从右往左读声明语句，《Effective C++》则建议你观察const与*的位置关系，两个含义其实一致——**const修饰距其最近的类型**。

```c++
const int* cVar = 0;

int* const cPtr = 0;

const int* const cc = 0;
```

> 从《C++ Prime》的角度分析：
>
> cVar是一个指针，它指向一个int常量；
>
> cPtr是一个常量指针，它指向一个int对象；
>
> cc是一个常量指针，它指向一个int常量。

但是存在一个特例，就是STL的迭代器，某些序列型容器，例如array和vector由于分配的是连续地址，其迭代器在实现上实际就是一个指针`T*`，但是使用const修饰迭代器对于它的理解不能是`const T*`而是`T* const`，即常量指针，表示这个迭代器不得指向不同的东西。假设需要一个`const T*`则需要使用const_iterator：

```c++
std::vector<int> vec;
...
const std::vector<int>::iterator iter = vec.begin(); // 迭代器本身是常量，不可修改
*iter = 10;		// 正确，迭代器指向int对象
iter++;			// 错误，迭代器是一个const

std::vector<int>::const_iterator cIter = vec.begin(); // 迭代器指向int常量
*cIter = 10;	// 错误
cIter++;		// 正确
```

这是因为虽然迭代器是一个`T*`，但事实上从STL的设计上来说迭代器其实是一个对象类型，这一点在其他容器(尤其是关联型容器)体现更加明显，因此我们应该把这个`T*`，也即迭代器本身当作一个整体来看，使用const修饰的正确理解应该是——**迭代器常量**。

而const_iterator在源码的实现上是一个poniter-to-const，它的所有返回都是const常量，相关的操作都是const成员函数。

**注：这里结论虽然和书中一致，但解释有所出入，是本人基于STL源码(GCC 2.9)实现的一点理解，假如有误还望指出。**

接下来书中提出了一个目前来说不合时宜的例子：

```c++
Rational a, b, c;
...
(a * b) = c;
// or
if (a * b = c) // 遗漏
```

且不说这类明显错误的出现概率，自从左值和右值的概念被提出来之后，这类错误已经能够被编译器识别，显然`a * b`作为一个函数返回不仅是临时变量还是一个非左值，我印象里即使不报error也会有warning提示(to-do: 实际测试)。

## const函数

将const实施于成员函数的目的，是为了确认该成员函数可作用于const对象。

- 明确函数能够生效的对象
- 使操作const对象成为可能——提高效率的一个根本方法是pass by reference-to-const，**因此我们经常需要处理由于函数参数传(passed by pointer-to-const/reference-to-const)递所引入的const对象**

### bitwise constness

> 成员函数只有在不更改对象之任何成员变量(static除外)时才可以说是const，也就是它不更改对象内的任何一个bit。

但是书中举了一个特殊的例子(我感觉不能深究书中例子)：

```c++
class CTextBlock {
public:
    ...
    char& operator[] (std::size_t position) const {
        return pText[position];
    }
private:
    char* pText;
}

// test
const CTextBlock cctb("Hello");
char* pc = &cctb[0];

*pc = 'J';
```

这个例子使用上的正误先不说，但确实展示了一种特殊的修改`const T*`的做法，在《C++ Prime》Chapter 2.4.2(Page 56)有提到：

> 所谓指向常量的指针或引用，不过时指针或引用“自以为是”罢了，它们觉得自己指向了常量，所以自觉地不去改变所指对象的值，而没有规定那个对象的值不能通过其他途径改变。

由于CTextBlock类的成员变量pText是一个指针，operater[]本身并不会更改pText，而外部修改pText所指对象而不是pText指针本身也不会引发编译器的错误。但我并不建议编写这样的代码，因为这样去修改值不符合直观认知，通常情况下更建议使用string对象而不是char*；其次通过const成员函数返回一个const对象的非const引用本身就是一种不安全的操作。

书中借此引出logical constness的概念。

### logical constness

> 一个const成员函数可以修改它所处理的对象内的某些bits，但只有在客户端侦测不出的情况下才得如此。

为了实现这点需要使用mutable释放掉non-static成员变量的bitwise constness约束。

## 在const和non-const成员函数中避免重复

书中不建议在程序中使用casting，确实C++并不像C语言一样支持隐式类型转换，但是当const和non-const成员函数有着实质等价的实现时，为了减少代码重复，在大部分的程序中，往往非const成员函数将调用const成员函数。

```c++
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const {
        return pText[position];
    }
    char& operator[] (std::size_t position) {
        return const_cast<char&>(static_cast<const TextBlock&>(
            		*this)[position]);
    }
...
}
```

**注：**这一item对const的使用提出了数点要求，但是恕本人才疏学浅，目前使用上仅局限于pass-by- reference-to-const和少量的const member function，并不会过于纠结const的细枝末节，事实上我觉得开发不应该被规则所束缚，对于你写的每一行代码做到心中有数即可。

# Item 4: 确定对象被使用前已被初始化

```c++
int x;
int y[100];
```

在大多数情况下(与编译器版本和语境有关)，在《C++ Prime》Chapter2.2.1(Page 40)中提到：

> 如果是内置类型的变量未被显式初始化，它的值由定义的位置决定。定义于任何函数体之外的变量被初始化为0。定义在函数体内部的内置类型变量将不被初始化。

但我在前段时间有做过一个整型情况下的测试，无论是单一变量还是数组，初值都是0。

**注：**无论怎么说读取未初始化的值会导致不明确的行为，将污染正在进行读取动作的那个对象，并不建议这么做。

本条例讨论了赋值和初始化两个概念，在《C++ Prime》Chapter2.2.1(Page 39)中对于两者有一个简单的解释：

> 初始化不是赋值，初始化的含义是创建变量时赋予其一个厨师值，而赋值的含义是把对象的当前值擦出，而以一个新值来替代。

C++规定，对象的成员变量的初始化动作发生在进入构造函数本体之前，因此更推荐的做法是使用初值列来完成初始化动作：

- 使用赋值语句完成初始化由两个动作组成：调用成员变量的默认构造函数设初值，然后再调用成员变量的拷贝构造函数赋新值
- **对于内置类型虽然其初始化和赋值的成本相同，但是用初值列来完成初始化会有更好的类型检查效果**
- 对于多构造函数类，可以抽提取出“赋值表现和初始化一样好”的成员变量，改用赋值操作并移往某个private函数
- C++的成员初始化顺序由成员变量声明顺序所决定

## Singleton meets C

接下来条例讨论了一个non-local static对象的初始化顺序问题，很不幸的是这个场景我并没有遇到过，因为我至少会确保所调用的对象一定在被调用时已被初始化。

不过接下来我想沿着Singleton这个主题讨论一个问题：不同编译单元内定义的同一个local static对象，这里的不同编译单元特指不同的DLL。

以我浅显的理解，设计模式是基于面向对象的，但作为一个Linux平台下的软件工程师，不可避免会使用显示调用的方式来加载指定DLL中的API，而这属于Linux C的内容，网上大部分文章只讨论了在C++程序中由于允许函数重载所导致的symbol mangling问题，这可以使用extern "C"来解决。但假如在这种情况下，我们引入local static对象，更明确一点，我们想要在不同的DLL中使用同一个Singleton对象，这将会发生什么？

想弄清楚这个问题，我们需要明确static关键字的作用，在《C++ Prime》Page 226的术语表中给出的解释是：

> local static object: 它的值在函数调用结束后仍然存在，在第一次使用局部静态对象前创建并初始化它，当程序结束时局部静态对象才被销毁。

**这里需要注意的是local static object初始化的时机**，书中的解释并不明显，因为静态变量的内存分配发生在编译阶段，或许我能在《程序员的自我修养》中找到严格定义，但当下先借鉴：

> local static object编译时在静态数据区分配内存，到程序结束时才释放。

local static object在编译阶段就会在静态段分配内存，但这也只是对于我们正在运行的程序而言的，当使用显示调用时，直到主程序加载DLL和API之前，DLL还未被加载进内存，就更不用讨论静态段了。于是当DLL被加载进，程序意识到这个DLL中也存在static对象，也会为其分配内存，这样就导致了**当我们使用显式调用加载多个DLL时，Singleton变成了一种伪单例，除了主程序外，每个DLL都拥有一个属于自己的单例对象。**



**注：**这个问题是我在调研日志系统，研究spdlog的单例模式时发现的，好在对于日志系统来说，我们还有一系列日志收集系统，它们能整合具有相同输出pattern的日志文件并重新以用户自定义的设置展示，至于其他情况，目前还未遇到过。

