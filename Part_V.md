# Item26: 尽可能延后变量定义式的出现时间

只要定义了一个变量并且其带有构造和析构函数，那么我们就必须要承担构造和析构成本，即使我们并不一定是用了这个变量(因为某些原因退出当前区块)，因此正确的做法是：**对于所有的变量，我们都需要以“具明显意义之初值”将变量初始化。**



```c++
// 定义于循环体外
Widget w;
for (int i = 0; i < n; i++) {
    w = ...
    ...
}

// 定义于循环体内
for (int i = 0; i < n; i++) {
    Widget w = ...
    ...
}
```

对于循环，两种写法的成本如下：

- 1个构造函数+1个析构函数+n个赋值操作
- n个构造和析构

如果class的一次赋值成本低于一次构造和析构，那么应该使用第一种写法，但由于第一种写法变量w的作用与被拓展，总是更加推荐第二种写法。

# Item27: 尽量少做转型操作

在C中有两种形式的类型转换操作，两种形式并无区别：

```c
(T)expression;
T(exporession);
```

在C++中提供四种命名的强制类型转换，具体介绍可以参考《C++ Prime》Chapter 4.11.3(Page 144)：

- `const_cast<T>(expression)`：通常被用来将对象的常量性移除，但即使移除了对象的常量性，执行写操作是一种未定义的行为。const_cast通常用于有函数重载的上下文中。
- `static_cast<T>(expression)`：任何具有明确定义的类型转换，只要不是移除常量性就能够使用static_cast完成，一种常见的用法是将void*具现化为原来的值。
- `reinterpret_cast<T>(expression)`：通常为运算对象的位模式提供较低层次上的重新解释，实际动作和结果可能取决于编译器。它几乎是一个万能的类型转换关键字，除非开发人员清楚的了解语句的行为及其后果，否则并不推荐使用它。
- `dynamic_cast<T>(expression)`：主要用来执行“安全向下转型”，例如base class向derived class转换。值得注意的是dynamic_cast速度非常慢。

强制类型转换干扰了正常的类型检查，除了特定的使用场景，我们应该保持这样一种认知：**使用强制类型转换往往意味着某些设计缺陷，我们总该思考是否可以使用其他方式实现相同的目标。**

书中给出了关于dynamic_cast的两种替代方法，一个是直接存储derived class指针，一个是使用虚函数，事实上我目前并没有见过必须要使用dynamic_cast的情景，多态足以解决大部分情况。

# Item28: 避免返回handles指向对象内部成分

这一条款的内容在Item 3的bitwise constness有类似的说明，主要讲述了通过返回non-const reference或pointer可能会导致类的封装性的破坏以及修改内部成员的一种可能性。

其实这里可以借用Item 21的说法：必须返回对象时，不要返回reference，虽然两者讨论的不是一个问题，但事实上这一条款讨论的成员函数返回reference才更有价值，因为我确信有很多人在getter函数的实现上返回reference。

# Item29: 为“异常安全”而努力是值得的

由于个人的私心，并不打算花精力在异常安全这一特性上，或许未来我转变了观念我会回过头来学习相关内容，目前我选择跳过这一条款。

# Item30: 透彻了解inlining的里里外外

这一条款阐述了inline的几种情形，但是讲的过于繁琐，且侯捷老师的翻译习惯开始让我感到不习惯，因此在这里我更想援引《C++ Prime》Chapter 6.5.2(Page 213)中的解释。虽然有些简单，但也足够了。

> 将函数指定为内联函数(inline)，通常就是将它在每个调用点上“内联地”展开。
>
> 内联说明只是向编译器发出的一个请求，编译器可以选择忽略这个请求。
>
> 一般来说，内联机制用于优化规模较小、流程直接、频繁调用的函数。很多编译器都不支持内联递归函数，而且一个75行的函数也不大可能在调用点内联地展开。

# Item31: 将文件间的编译依存关系降至最低

这一条款旨在解决这样一个问题：在编译过程中尽可能只编译发生修改的部分，其余的工作交给连接器。要实现这样的目标，需要我们将接口从实现中分离，否则在编译的时候将重编所有与修改有关的内容。

一种常见的实现手段就是采用pointer to implementation class，将所有关于这个类(Handle class)的请求都委派给真正的实现类，这种实现能够将“接口与实现分离”：隐藏实现细节，接口具有更大的拓展弹性，只修改实现及其依赖的时候不会影响接口的使用，即不会影响到客户代码。

另一种实现手段是使用抽象类(Interface class)，在Item 9反驳书中样例的时候我曾提到抽象类的作用：对于抽象类的简单理解就是它无法实例化，于是这个类纯粹是为了提供接口的，这意味着对于纯虚函数的实现我们交由derived class定制，而原则上抽象类自身并不会实现纯虚函数。用户通常借由一个工厂函数来具像化derived class的对象(返回一个std::shared_ptr)，这同样向客户屏蔽了接口的具体实现。

书中最后阐述了两种实现的开销，使用handle class，成员函数必须通过implementation pointer完成操作，每一个handle class对象内存将增加一个implementation pointer大小，以及implementation pointer初始化和释放的成本。而使用Interface class，由于virtual函数的实现，除了会引入一张虚函数表的内存开销，每一次调用其实都会需要做一次offset，也就是增加一次间接跳跃成本。

最后书中从Item 23开始就在强调的，关于程序实现代码布局上的一点建议：为声明式和定义式提供不同的头文件。