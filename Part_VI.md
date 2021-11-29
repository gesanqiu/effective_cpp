本章节最有价值的部分在于解释了C++各种不同特性的真正意义，假如你熟悉侯捷老师的C++课程，那么你应该知道侯捷老师一直强调的C++类间关系：Composition，Delegation，Inheritance以及virtual/non-virtual，这些内容是OOP的基础，也是八股文的重要考点。

此外本章节还讨论了OOP中继承这一重要特性在不同的场景下的各种用法，个人浅见无论是OOP还是更加深奥的设计模式，这一切都是在继承这一理论基础上发展的，作为一个开发人员能够探究继承的各种细枝末节对于个人总是有好处的。

# Item32: 确定你的public继承塑模出is-a关系

> 如果你令class D(Derived)以public形式继承class B(Base)，你便是告诉C++编译器(以及你的代码读者)说，每一个类型为D的对象同时也是一个类型为B的对象，反之不成立。你的意思是B比D表现出更一般化的概念，而D比B变现出更特殊化的概念。你主张“B对象可以派上用场的任何地方，D对象一样可以派上用场”，因此每一个D对象都可以是一个B对象。反之如果你需要一个D对象，B对象无法效劳，因为虽然每个D对象都是一个B对象，反之并不成立。

在最初学习C++的时候，对于public继承我有一个错误的认知：从数据(成员变量和成员函数)的拓展性角度，我们很容易想到数学中的“集合”模型，进而得出父类属于子类这样的结论，然后参考集合的特性——对某个集合成立的结论，对其子集也总是成立，我将得到一个与public继承本意相反的结论。

这只是一种直觉，这种抽象完全违背了public继承的本意：**子类是一种特殊的父类，也即所谓的is-a关系**。假如我们从功能的角度出发，那么就可以知道由于父类的所有内容都将被子类继承，因此对所有父类的操作对子类也成立(这其实也是多态的理论基础)。

在此基础上，本条款进行了更加深入的探讨：在使用public继承时，**子类必须涵盖父类的所有特点，必须无条件继承父类的所有特性和接口**。

```c++
class Bird {
public:
    virtual void fly();
    ...
};

class Penguin: public Bird {
    ...
};

//------ -----compile error -------------//
class Bird {
    ...
};

class FlyingBird {
public:
    virtual void fly();
};

class Penguin: public Bird {
    ...				// no fly() function declaration
};

//-------------runtime error-------------//
class Bird {
public:
    virtual void fly();
    ...
};

class Penguin: public Bird {
public:
    virtual void fly() { error("Attemp to make a penguin fly!"); }
};
```

上述例子就“fly”这一接口讨论了企鹅和鸟之间的public继承关系，很明显企鹅是一种鸟，但企鹅也确实不会飞，这个例子说明：在设计继承的时候会掉进一些由于认知和语言导致的陷阱中。书中给出了两种解决思路：编译阶段报错(禁用企鹅的fly接口)和运行时报错(在发生调用时进行相应的错误处理)，这个问题虽然在Item 18有过讨论，但我认为它并没有一个统一的结论，几乎完全取决于开发者对于程序的认知以及实际需求。

当然假如你对鸟的关注点不在fly而在某些更为commom的特性上，企鹅也完全可以public继承鸟类，这反映了一个事实：**世界上并不存在一个“适用于所有软件”的完美设计。所谓最佳设计，取决于系统希望做什么事，包括现在和未来。**

# Item33: 避免遮掩继承而来的名称

```c++
class Base {
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
    ...
private:
    int x;
};

class Derived: public Base {
public:
    // using Base::mf1;		// 让Base class内名为mf1和mf3的所有东西
    // using Base::mf3;		// 在Derived作用域内可见
    virtual void mf1();
    void mf3();
    void mf4();
    ...
};

Derived d;
int x;
...
d.mf1();
d.mf1(x);		// 错误，Derived::mf1遮掩了Base::mf1
d.mf2();
d.mf3();
d.mf3(x);		// 错误，Derived::mf3遮掩了Base::mf3
```

不加探究例子合理性的，本条款想要阐述作用域遮盖在继承中的情况：名称在作用域级别的遮盖与参数类型和是否virtual函数无关，从上面的例子可以看出，**所有的**Base::mf1和Base::mf3都不再被Derived继承。假如想要重启父类中被遮掩的函数，那么你必须为这些名称引入一个using声明式。

# Item34: 区分接口继承和实现继承

我们在条款32讨论了public继承的实际意义，我们在本条款将明确在public继承体系中，不同类型的接口——纯虚函数、虚函数和非虚函数——**背后隐藏的设计逻辑**。

首先需要明确的是，成员函数的接口总是会被继承，而public继承保证了，如果某个函数可施加在父类上，那么他一定能够被施加在子类上。不同类型的函数代表了**父类对子类实现过程中不同的期望**。

- 在父类中声明纯虚函数，是为了**强制子类拥有一个接口**，并**强制子类提供一份实现**。
- 在父类中声明虚函数，是为了**强制子类拥有一个接口**，并**为其提供一份缺省实现**。
- 在父类中声明非虚函数，是为了**强制子类拥有一个接口以及一份强制性的实现**，并不允许子类对其做任何更改。

**以上的种种，其实都是围绕着函数的不变形和特定性进行讨论。**

书中在阐述impure virtual函数时指出其缺省实现将造成危险：

```c++
class Airport { ... };
class Airplane {
public:
    virtual void fly(const Airport& destination);
    ...
};
void Airplane::fly(const Airport& destination)
{
    缺省实现，将飞机飞至指定的目的地
}
class ModelA: public Airplane { ... };
class ModelB: public Airplane { ... };

class MOdelC: public Airplane { ... };


//----------------pure virtual function------------------//
class Airplane {
public:
    virtual void fly(const Airport& destination);
    ...
protected:
    void defaultFly(const Airport& destination)		// 没有任何derived class应该重新定义这个函数
};
void Airplane::defaultFly(const Airport& destination)
{
    缺省实现，将飞机飞至指定的目的地
}
class ModelA: public Airplane {
public:
    virtual void fly(const Airport& destination) {
        defaultFly(destination);
    }
};
class ModelB: public Airplane {
    virtual void fly(const Airport& destination) {
        defaultFly(destination);
    }
};
// 由于被声明为pure virtual函数，迫使C型飞机必须提供自己的fly版本
class ModelC: public Airplane {
    virtual void fly(const Airport& destination);
    ...
};
void ModelC::fly(const Airport& destination)
{
    将C型飞机飞至指定的目的地
}
```

> 在这其中，有可能出现问题的是普通虚函数，这是因为父类的缺省实现并不能保证对所有子类都适用，因而当子类忘记实现某个本应有定制版本的虚函数时，父类应从__代码层面提醒子类的设计者做相应的检查__，很可惜，普通虚函数无法实现这个功能。一种解决方案是，在父类中**为纯虚函数提供一份实现**，作为需要主动获取的缺省实现，当子类在实现纯虚函数时，检查后明确缺省实现可以复用，则只需调用该缺省实现即可，这个主动调用过程就是在代码层面提醒子类设计者去检查缺省实现的适用性。

这是关于这个例子的一种理解，但是我觉得没有必要盲目相信权威，以下是我对于这个例子的一些思考：

- 书中提到“为了避免在ModelA和ModelB中**撰写相同代码**，缺省飞行行为由Airplane::fly提供”，这个撰写相同代码我有两种理解，一是Airplane类的缺省代码即可满足ModelA和ModelB的fly需求，但是在这种情况下，我们甚至不需要将fly定义为virtual；二是ModelA和ModelB的fly依赖于Airplane::fly，那这种情况下为什么不直接采用类似于defaultFly的方式，将Airplane::fly声明为一个非虚成员函数，依然是是否有必要使用virtual函数(当然这是Item 35将要讨论的问题)。
- 在引入ModelC时，“由于它们着急让心飞机上线服务，竟忘了重新定义其fly函数”，解决方法是将fly声明为pure virtual函数，确保所有的derived class都有自己实现，从我的理解来看这部分应该是pure virtual函数的特性，在编排上并不应该出现在impure virtual里，尤其是考虑到前文侯捷老师反驳pure virtual的实用性，我不是很能理解这种叙述。
- 上文中“父类的缺省实现并不能保证对所有子类都适用”，既然不能够保证缺省的实现对所有子类都适用，那么为什么要给virtual函数提供缺省实现。或许是考虑到未来维护者或其他开发者并不知道设计的初衷，但是我并不认为要将所有的开发责任都一股脑抛给作者，程序作者在某种程度上还是一个制定使用规则的人，后来者应该在遵守规则的基础上进行工作，工作的边界是现实开发中必须要考虑的问题。
- 最后书中就“有些人反对以不同的函数分别提供接口和缺省实现”，进行了补充，如我前文中所强调的一般，我觉得我们需要区分接口和实现，引入pure virtual函数之后的抽象类，永远只是为了提供接口而存在的，我十分反对给pure virtual函数提供缺省实现，这并不是一种灵活的实现方式，你丧失了一部分的权限控制的权利。

# Item35: 考虑virtual函数以外的其他选择

这一条款讨论的是virtual函数的替代方案，提到virtual函数，我们的关注点其实是其动态绑定的特性，也即运行时绑定，这一条款阐述了几种不同的运行时绑定方式。

## 藉由Non-Virtual Interface首发实现Template Method

## 藉由std::function完成Strategy模式

## 古典的Strategy模式

# Item36: 绝不重新定义继承而来的non-virtual函数

```c++
class B {
public:
    void mf();
    ...
};

class D: public B {
public:
    void mf();
    ...
};

D x;
B* pB = &x;
pB->mf();		// 调用B::mf
D* pD = &x;
pD->mf();		// 调用D::mf
```

这一条款讨论的依然是动态绑定和静态绑定相关的概念，造成上述不清晰的调用行为的原因在于non-virtual函数是静态绑定的，它的静态类型是什么那么就将调用哪个类型的函数，因此假如我们有动态绑定需求的时候一定要使用virtual函数，在C++11中可以使用final关键字确保接口不被继承覆盖。

# Item37: 绝不重新定义继承而来的缺省值

> virtual函数是动态绑定，而缺省参数值是静态绑定，这意味着你可能会在“调用一个定义于derived class内的virtual函数”的同时，却使用base class为它所制定的缺省参数值，因为子类对缺省参数作出的改变并不会生效。

缺省参数值属于静态绑定的原因是为了提高运行时效率。

如果你真的想让某一个虚函数在这个类中拥有缺省参数，那么就把这个虚函数设置成private，在public接口中重制非虚函数，让非虚函数这个“外壳”拥有缺省参数值，当然，这个外壳也是一次性的——在被继承后不要被重载。

# Item38: 通过复合塑模出has-a关系或“根据某物实现出”

至此三个类间关系已全部被引出，复合(Composition)意味has-a(有一个)或is- implemented-in-terms-of(根据某物实现出)，根据将要处理的领域进行区分：

- 程序中的对象其实相当于你所塑造的世界中的某些事物，例如Address，PhoneNumber这样的类对象属于应用域的部分。has-a表示一个类拥有另外一个类的对象作为一个成员，当复合发生在应用域的对象之间，表现出has-a的关系。
- 另外的对象可能是实现细节上的人工制品，例如Mutex，Timer这样的类对象往往是用来提供某些基础操作，我们可以基于它们进行封装，这些对象属于实现域的部分。is- implemented-in-terms-of表示的是用一个工具类去实现另一个类，这种情况下将隐藏工具类，只对外暴露接口，例如在STL中queue就是基于deque对象实现的

复合的is- implemented-in-terms-of关系在功能上有点类似于之前提到的Delegation(pointer to implement)关系，两者的区别是一个是对象，一个是指针。

# Item39: 明智而审慎地使用private继承

private继承：

- 如果classes之间的继承关系是private，编译器不会自动将一个derived calss对象转换为一个base class对象。
- 由private base class继承而来的所有成员，在derived class中都会变成private属性。

> private继承意味着只有实现部分被继承，接口部分应略去。private继承在软件“设计”层面上没有意义，其意义只及于软件实现层面。

private继承以为implemented-in-terms-of，上述引用表述的意思是base class的成员由于private继承在derived class中全部变为private属性，因此base class的内容只在derived class内部可见，接口甚至不向外部暴露，这代表了一种实现上的封装。

书中接下来讨论了同样可以表示implemented-in-terms-of含义的复合和private继承之间的取舍问题：**尽可能使用复合，除非必要，否则不要使用private继承，而且几乎总是可以想到private的替代方案。**

```c++
class Timer {
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
    ...
};

class Widget: private Timer {
private:
    virtual void onTick() const;
    ...
};

//---------------------------------------------------------//
class Widget: private Timer {
private:
    class WidgetTimer: public Timer {
        virtual void onTick() const;
        ...
    };
    WidgetTimer timer;
    ...
};
```

Timer是一个可以以一定时钟频率运行的定时器，Widget每个一定时间调用onTick用以记录数据。样例代码展示了两种实现方式，private继承通过虚函数可以实现自定义onTick接口而又避免了public继承is-a关系的矛盾，但这其实并不安全，在书籍出版的06年由于没有final关键字，因此不能阻止Widget的derived class继续改写WidgetTimer的接口。而基于复合首发除了可以防止后续复写还能降低Widget的编译依存性，STL中就大量使用了这样的实现设计。

# 条款40：明智而审慎地使用多重继承

> 原则上不提倡使用多继承，因为多继承可能会引起多父类共用父类，导致在底层子类中出现多余一份的共同祖先类的拷贝。为了避免这个问题C++引入了**虚继承**，但是虚继承会使子类对象变大，同时使成员数据访问速度变慢，这些都是虚继承应该付出的代价。
>
> 在不得不使用多继承时，请慎重地设计类别，尽量不要出现菱形多重继承结构（“B、C类继承自A类，D类又继承自B、C类”），即尽可能地避免虚继承，一个完好的多继承结构不应在事后被修改。虚基类中应尽可能避免存放数据。

与其说原则上不提倡使用多继承，我更加愿意禁掉多重继承这种设计，有可能的话它甚至不应该出现在八股文中，话虽如此，多继承一直是我工作以来的的一个结。阅读《Effective C++》的整个过程中，我曾对许多例子都提出了批判，但多继承是唯一一个我熟悉却又非常不熟悉的知识点，熟悉是因为我曾花了大量时间去研究虚函数表，并且试图找到多重继承和菱形继承的应用场景，不熟悉就在于我根本没遇到过多继承的情况，并且菱形继承无疑是一种反直觉的设计，处处违背了is-a的原则，或许若干年后我会回过头来补充关于多继承的理解，但目前关于它的讨论就仅限于此。