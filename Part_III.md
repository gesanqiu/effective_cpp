# Item13: 以对象管理资源

与拥有垃圾回收机制的语言不通，C++给予了开发者足够的自由。所谓资源就是一旦用了它，将来必须还给系统。新手往往会陷入一个误区，认为资源就是内存，但广义上的资源包括一切请求，例如文件描述符，互斥锁，数据库连接，网络sockets等。

```c++
class Investment { ... };

Investment* createInvestment();		// 工厂方法

void f()
{
    Investment* pInv = createInvestment();
    ...
    delete pInv;
}
```

以上操作在大多数情况下总是正确的，但是这种管理方式意味着资源的索取者肩负着释放它的责任，于是书中提出了几点合理的疑问：

- 一句简单的`delete`语句并不会一定执行，例如一个过早的`return`语句或是在`delete`语句之前某个语句抛出了异常。
- 谨慎的编码可能能在这一时刻保证程序不犯错误，但无法保证软件接受维护时，其他人在delete语句之前加入的return语句或异常重复第一条错误。

> 为确保createInvestment()返回的资源总是被释放，我们需要将资源放进对象内，当控制流离开f，该对象的析构函数会自动释放哪些资源。

`auto_ptr`就是为这种场景设计的类指针对象，其析构函数自动对其所指对象调用delete：

```c++
void f()
{
    std::auto_ptr<Investment> pInv(createInvestment());
    ...
}
```

- 获得资源后立即放进管理对象：createInvestment返回的资源立马成为其资源管理者auto_ptr的初值。
- 管理对象运用析构函数确保资源被释放：无论控制流如何离开区块，一旦对象被销毁（比如离开对象的作用域）其析构函数会自动被调用。

需要注意的是多个auto_ptr不能指向同一对象，否则由于其被销毁是会调用析构函数的特点，这个对象会被删除多次，将导致未定义行为，因此auto_ptr实际仅用了copying函数。

书中当是由于还没有`= delete`关键字，所以auto_ptr对于指向的资源具有唯一拥有权。

当然更加推荐的做法是使用`std::shared_ptr`——引用计数型智能指针。

```c++
void f()
{
    std::shared_ptr<Investment> pInv1(createInvestment());
    
    std::shared_ptr<Investment> pInv2(pInv1);
    pInv1 = pInv2;		// pInv1和pInv2指向同一对象
    ...

}
```

# Item14: 在资源管理类中小心copying行为

条款13导入这样的观念：“资源取得时机便是初始化时机(Resource Acquisition Is Initialization; RAII)”，并以此作为“资源管理类”的记住，也描述了auto_ptr和shared_ptr如何将这个观念表现在heap-baed资源上。但如前文所说，资源不仅仅是memory，还有可能是互斥锁着类非heap-based资源，这时候智能指针将会失效，于是开发人员需要实现自己的资源管理类。

```c++
class Lock {
public:
    explicit Lock(Mutex* pm) : mutexPtr(pm) {
        lock(mutexPtr);
    }
    ~Lock() { unlock(mutexPtr) };
private:
    Mutex* mutexPtr;
};
```

书中举例为确保不会忘记将一个被锁住的Mutex解锁，建立了一个Lock类来管理Mutex，符合RAII守则，即“资源在构造期间获得，在析构期间释放”。

```c++
Mutex m;
...
{	// 一个自定义区块
    Lock m1(&m);	// 上锁
    ...				// 区块操作
}	// 解锁
```

在正确的操作下，几乎不会遇到什么问题，但是假如我们尝试去复制Lock对象，那么会发生什么：

```c++
Lock m1(&m);
Lock m2(m1);
```

- 对于互斥锁对象来说，复制一个互斥锁是一个不可能的行为，因此我们应该使用`= delete`来禁止显式声明copying函数。
- 但是在某些特殊情况下，我们希望保有资源直到它的最后一个使用者被销毁，引用计数就发挥了强大的作用。

引用计数实际被应用在很多场景中，因为拷贝的开销总是令人难以接受的，所以往往大部分C++开源库的拷贝操作都会优先考虑浅拷贝，即增加引用计数，对于深拷贝提供单独的接口。

在这个例子中我们可以修改mutexPtr为一个shared_ptr对象以达到这个效果：

```c++
class Lock {
public:
    explicit Lock(Mutex* pm) : mutexPtr(pm, unlock) {
        lock(mutexPtr.get());
    }
private:
    std::shared_ptr<Mutex> mutexPtr;
};
```

需要注意的是这个例子中不再声明析构函数，因为编译器默认生成的析构函数就已经足够了——Lock对象在析构时会自动调用mutexPtr的删除器，而mutexPtr的删除器绑定了unlock操作。

**注：**在C++11之后已经有`std::unique_lock`和`std::lock_guard`实现了相同的功能。

# Item15: 在资源管理类中提供对原始资源的访问

本条款的核心在于虽然资源管理类很好，能够避免资源泄漏，但是在实际开发中，我们往往需要直接获得原始资源的操作权，以便对资源进行一系列读写操作，于是一个资源管理类需要对外提供相应的接口。

- 智能指针提供了一个get成员函数，用来执行显式转换，返回智能指针内部的原始指针的副本。
- 智能指针同时重载了operator->和operator*操作符用以隐式转换至底部原始指针。

书中后续讨论了自定义资源管理类中显式转换和隐式转换的区别，但我一律建议使用显式转换，因为隐式转换实在过于晦涩，并且这种不明显的行为在出错时会导致巨大的调试成本。

## Item16: 成对使用new和delete时要采取相同形式

关于这一条款，我有些过一篇详细的文章，分析了new和delete的不同组合下的具体行为：[C++中的动态内存分配：new和delete](https://zhuanlan.zhihu.com/p/415093525)，这里仅简单分析。

> delete的最大问题在于：即将被删除的内存之内究竟存有多少对象？这个问题的答案决定了有多少个析构函数必须被调用起来。

- 数组所用的内存通常还包括“数组大小”的记录，但这个记录是否生成取决于对象是否有显式声明的析构函数。
- 根据内存布局和delete的汇编代码可以知道，对于delete[]而言，会去pointer的前4/8字节以获取关于“数组大小”的记录。
- 书中提到的typedef数组对象，个人并不建议这种行为，会增加代码阅读难度以及导致未定义的的行为。

**注：**尽管在我的分析文章中讲述了一些特殊情况，但是还是应该严格遵守语法规范，采取相同的形式。

# Item17: 以独立语句将newed对象置入智能指针

这一条款的核心在于：“资源被创建”和“资源被转换为资源管理对象”两个时间点之间有可能发生异常。

```c++
int priority();		// 程序处理优先级函数
void processWidget(std::shared_ptr<Widget> pw, int priority);

processWidget(std::shared_ptr<Widget>(new Widget), priority());
```

 由于C++编译器对于函数调用，传递非实参时的调用顺序并没有严格的规定，对于上述`processWidget`调用只能确保new Widget在智能指针对象构造之前调用，并不能确定priority()的调用时机。因此考虑如下调用顺序

1. 执行new Widget
2. 调用priority()
3. 调用std::shared_ptr构造函数

假如priority()调用失败，new Widget指针将会丢失，造成资源泄漏。



