# 一、关于对象

- C 语言是**程序性**的，语言本身并没有支持数据和函数之间的**关联性**
- C++ 中可能采取**抽象数据类型**，或者是**多层次的类**结构完成
- C++ 的封装并没有增加多少成本，每一个[成员函数](https://so.csdn.net/so/search?q=成员函数&spm=1001.2101.3001.7020)虽然在class中声明，但是却不出现在每个对象中
  - 每一个**非内联**的成员函数只会诞生**一个**函数实例
  - 每个内联函数会在其**每一个**使用者身上产生一个函数实例
- C++ 在布局以及存储时间上主要的额外负担是由**virtual**引起的
  - 虚函数机制用以支持一个有效率的“**执行期绑定**”
  - 虚基类用来实现“**多次出现在继承关系中的基类，有一个单一而被共享的实例**”
- 还有一些多重继承下的额外负担，发生在**一个派生类和其第二或后继之基类的转换**之间

## 1.1：C++对象模式

​	C++中有两种成员变量：静态和非静态；有三种成员函数：静态、非静态和虚函数

**简单对象模型**

<img src="https://pic4.zhimg.com/v2-77fc1d12af959990f0f3b5a4c70bef9b_b.jpg" alt="img" style="zoom:67%;" />

​	在这个简单模型下member本身并不放在object之中，只有指向member的指针才放在object内。这么做可以避免member有不同的类型，因而不需要不同的存储空间导致的问题。<u>这个模型并没有被应用到产品之中。</u>

**表格对象驱动模型**

<img src="https://pic2.zhimg.com/v2-4ec5c05bee78535a344f55bee62bf919_b.jpg" alt="img" style="zoom: 25%;" />

​	数据与函数分离。object 为指向两表的两个指针。

**C++对象模型**

<img src="https://pic2.zhimg.com/v2-97a6ae64fbbe6c0405e4e9209419f7a9_b.jpg" alt="img" style="zoom:80%;" />

- Nonstatic data members 在每个class object之内
- static data members 在class object之外
- static 和 Nonstatic function members被放在类对象之外。
- 虚函数不同：

  - 每个类中存放一个指针称为**vptr**，指向**虚函数表**
  - 表中每个都指向一个虚函数
- 非静态成员改变则需要全部重新编译。优点空间和时间效率高

## 1.2：关键词所带来的差异

- `int ( *pq ) ( ); //声明`
- 当语言无法区分那是一个声明还是一个表达式时，我们需要一个超越语言范围的规则，而该规则会将上述式子判断为一个“声明“
- struct和class可以相互替换，他们只是默认的**权限不一样**
- 如果一个程序员需要拥有C声明的那种struct布局，可以抽出来**单独**成为struct声明，并且和C++部分**组合**起来

```c++
struct C_poinit {...};

class Point{
public:
    operator C_point(){ return _c_point };
    //...
private:
    C_point _c_point;
}
```

## 1.3：对象的差异

​	C++支持三种程序设计范式

- 程序模式。和C一样
- 抽象数据类型模型，加入类
- 面向对象模型，加入多态

​

C++以下列方法支持多态：

1. 经由一组隐式的转化操作。例如把一个 derived class 指针转化为一个指向其 public base type的指针。
2. 经由 virtual function机制。
3. 经由 dynamic_cast和 typeid运算符。

​	多态的主要用途是经由一个共同的接口来影响类型的封装，这个接口通常被定义在一个抽象的base class中。

需要多少内存才能够表现一个class object？一般而言要有：

- 其 nonstatic data members的总和大小。
- 加上任何由于 alignment（译注）的需求而填补（padding）上去的空间（可能存在于 members之间，也可能存在于集合体边界）。译注：alignment就是将数值调整到某数的倍数。在32位计算机上，通常alignment为4 bytes（32位），以使bus的“运输量”达到最高效率。
- 加上为了支持 virtual而由内部产生的任何额外负担（overhead）。

![img](https://pic4.zhimg.com/v2-e154e66ff7863af466278e4796f50433_b.jpg)

“指针类型”会教导编译器如何解释某个特定地址中的内存内容及其大小。

总而言之，多态是一种威力强大的设计机制，允许你继承一个抽象的public接口之后，封装相关的类型。需要付出的代价就是额外的间接性——不论是在“内存的获得”或是在“类型的决断”上。C++通过class的pointers和references来支持多态，这种程序设计风格就称为“面向对象”。

C++也支持具体的ADT程序风格，如今被称为object-based（OB）。例如String class，一种非多态的数据类型。String class可以展示封装的非多态形式；它提供一个public 接口和一个private实现品，包括数据和算法，但是不支持类型的扩充。一个OB设计可能比一个对等的OO设计速度更快而且空间更紧凑。速度快是因为所有的函数调用操作都在编译时期解析完成，对象建构起来时不需要设置 virtual机制；空间紧凑则是因为每一个class object 不需要负担传统上为了支持virtual机制而需要的额外负荷。不过，OB设计比较没有弹性。

# 二、构造函数语意学

## 2.1：default constructor

**“带有 Default Constructor”的 Member Class Object“**

当编译器需要的时候，会为类合成一个 default constructor，但这个被隐式声明的默认构造函数是不太好的。

如果一个class没有任何constructor，但它内含一个member object，而后者有default constructor，那么这个class的implicit default constructor就是“nontrivial”，编译器需要为该class 合成出一个default constructor。不过这个合成操作只有在constructor真正需要被调用时才会发生。再一次请你注意，被合成的default constructor只满足编译器的需要，而不是程序的需要。

**不同编译模块如何避免合成生成了多个默认构造函数？**

解决方法是以 inline 的方式合成，inline 函数具有静态链接，不会被文件以外看到。如果函数太复杂，会合成一个 explicit non-inline static 实例。

**有多个 member objects 要求初始化怎么办？**

按 members 在 class 里的声明顺序进行初始化。

如果写了一个构造函数，而没有把类实例成员变量初始化的话，这个构造函数会被编译器扩充从而把这些类实例成员变量也初始化了

**“带有 Default Constructor”的 Base Class**

如果一个没有任何constructors的class派生自一个“带有default constructor”的base class，那么这个derived class 的default constructor 会被视为nontrivial，并因此需要被合成出来。它将调用上一层 base classes 的 default constructor（根据它们的声明顺序）。对一个后继派生的class而言，这个合成的constructor和一个“被显式提供的default constructor”没有什么差异。

**“带有一个 Virtual Function”的 Class**

另有两种情况，也需要合成出default constructor：

1. class声明（或继承）一个 virtual function。
2. class派生自一个继承串链，其中有一个或更多的 virtual base classes。

**“带有一个 Virtual Base Class”的 Class**

Virtual base class 的实现法在不同的编译器之间有极大的差异。然而，每一种实现法的共同点在于必须使virtual base class在其每一个derived class object中的位置，能够于执行期准备妥当。

有4种情况，会造成“编译器必须为未声明 constructor 的classes合成一个default constructor”。C++Standard 把那些合成物称为 implicit nontrivial default constructors。被合成出来的constructor只能满足编译器（而非程序）的需要。它之所以能够完成任务，是借着“调用member object或base class的default constructor”或是“为每一个object初始化其virtual function机制或virtual base class机制”而完成的。至于没有存在那4种情况而又没有声明任何constructor的classes，我们说它们拥有的是implicit trivial default constructors，它们实际上并不会被合成出来。

在合成的 default constructor 中，只有 base class subobjects 和 member class objects会被初始化。所有其他的nonstatic data member（如整数、整数指针、整数数组等等）都不会被初始化。这些初始化操作对程序而言或许有需要，但对编译器则非必要。如果程序需要一个“把某指针设为0”的default constructor，那么提供它的人应该是程序员。

C++新手一般有两个常见的误解：

1. 任何class如果没有定义default constructor，就会被合成出一个来。
2. 编译器合成出来的default constructor会显式设定“class 内每一个 data member的默认值”。

## 2.2：copy constructor

三种情况会以一个对象的内容作为另一个对象的初值

1. 显示的初始化操作
2. 作为参数传递
3. 作为返回值传回类对象

当class object 以“相同 class 的另一个 object”作为初值，其内部是以所谓的default memberwise initialization手法完成的，也就是把每一个内建的或派生的data member（例如一个指针或一个数组）的值，从某个object拷贝一份到另一个object身上。不过它并不会拷贝其中的 member class object，而是以递归的方式施行 memberwise initialization（深拷贝）。

C++Standard上说，如果class没有声明一个copy constructor，就会有隐式的声明（implicitly declared）或隐式的定义（implicitly defined）出现。和以前一样，C++Standard 把copy constructor区分为trivial和nontrivial两种。只有nontrivial的实例才会被合成于程序之中。决定一个copy constructor是否为trivial的标准在于class 是否展现出所谓的“bitwise copy semantics”。

Memberwise copy(深拷贝): 在初始化一个对象期间，基类的构造函数被调用，成员变量被调用，如果它们有构造函数的时候，它们的构造函数被调用，这个过程是一个递归的过程。

Bitwise copy(浅拷贝): 原内存拷贝。例子：给定一个对象 object，它的类型是 class Base。对象 object 占用 10 字节的内存，地址从 0x0 到 0x9。如果还有一个对象 objectTwo ，类型也是 class Base。那么执行 objectTwo = object; 如果使用 Bitwise 拷贝语义,那么将会拷贝从 0x0 到 0x9 的数据到 objectTwo 的内存地址，也就是说 Bitwise 是字节到字节的拷贝。

只有在默认行为所导致的语意不安全或不正确时，我们才需要设计一个 copy assignment operator。默认的 memberwise copy 行为对于我们的 Point object 不安全吗? 不正确吗? 不，由于坐标都内含数值，所以不会发生"别名化（aliasing）"或"内存泄漏（memory leak）"。如果我们自己提供一个 copy assignment operator，程序反倒会执行得比较慢。

如果我们不对 Point 供应一个 copy assignment operator，而光是仰赖默认的 memberwise copy，编译器会产生出一个实例吗? 这个答案和 copy constructor 的情况一样∶实际上不会! 由于此 class 已经有了 bitwise copy 语意，所以 implicit copy assignment operator 被视为毫无用处，也根本不会被合成出来。

一个 cass 对于默认的 copy assignment operator，在以下情况，不会表现出 bitwise copy 语意∶

1. 当 class 内含一个 member object，而其 class 有一个 copy assignment operator 时。
2. 当一个 class 的 base class 有一个 copy assignment operator 时。
3. 当一个 class 声明了任何 virtual functions（我们一定不要拷贝右端 class object 的 vptr 地址，因为它可能是一个 derived class object）时。
4. 当 class 继承自一个 virtual base class（不论此 base class 有没有 copy operator）时。

重新设定Virtual Table的指针

回忆编译期间的两个程序扩张操作（只要有一个class声明了一个或多个virtual functions就会如此）：

- 增加一个virtual function table（vtbl），内含每一个有作用的virtual function的地址。
- 一个指向virtual function table的指针（vptr），安插在每一个class object内。

合成出来的ZooAnimal copy constructor 会显式设定object的vptr指向ZooAnimal class的virtual table，而不是直接从右手边的class object中将其vptr现值拷贝过来。

处理 Virtual Base Class Subobject

Virtual base class的存在需要特别处理。一个class object 如果以另一个object作为初值，而后者有一个 virtual base classsubobject，那么也会使“bitwise copy semantics”失效。

每一个编译器对于虚拟继承的支持承诺，都代表必须让“derived class object中的virtual base class subobject位置”在执行期就准备妥当。维护“位置的完整性”是编译器的责任。“Bitwise copy semantics”可能会破坏这个位置，所以编译器必须在它自己合成出来的copy constructor中做出仲裁。

我们已经看过4种情况，在那些情况下class不再保持“bitwise copy semantics”，而且 default copy constructor 如果未被声明的话，会被视为nontrivial。在这4种情况下，如果缺乏一个已声明的copy constructor，编译器为了正确处理“以一个class object 作为另一个class object 的初值”，必须合成出一个copy constructor。

## 2.3：程序转化

显式的初始化操作：重写每一个定义，其中初始化操作被剥除，class的拷贝构造被安插进去

参数的初始化：void foo(X x0); 被转化为先产生临时对象，x0内容拷贝给temp，最后函数用的是temp

返回值初始化：返回局部对象，则首先加上一个引用参数，用来放置返回值；然后把局部对象拷贝给此参数

## 2.4：成员们的初始化队伍

- 四种情况下你需要使用成员初始化列表
  - **当初始化一个引用成员变量**
  - **当初始化一个const 成员变量**
  - **当调用一个基类的构造函数，而它拥有一组参数**
  - **当调用一个类成员变量的构造函数，而它拥有一组参数**

```c++
class Word{
	String _name;
	int _cnt;
public:
	Word(){
		_name = 0;
		_cnt = 0;
	}
/*使用成员列表初始化可以解决    
    	Word() : _name(0)，_cnt(0){

	}
*/	
}
```

> 上式不会报错，但是会有效率问题，因为**这样会先产生一个临时的string对象，然后将它初始化，之后以一个赋值运算符将临时对象指定给_name，再摧毁临时的对象**
>

# 三、Data语意学

```c++
以下是gcc运行结果
class X{};
class Y : public virtual X {};
class Z : public virtual X {};
class A : public Y,public Z {};

sizeof(X)	//1
sizeof(Y)	//8 
sizeof(Z)	//8
sizeof(A)	//16  一个虚基类只会存在一份实例
```

- X为1是因为编译器的处理，在其中插入了1个char，为了**让其对象能在内存中有自己独立的地址**
- Y，Z**是因为虚基类表的指针加上x的一个，最后为了对其所以是8**
- **A 中含有Y和Z所以是16**
- 每一个类对象大小的影响因素：
  - **非静态成员变量的大小**
  - **virtual特性**
  - **内存对齐**

## 3.1：Data Member的绑定

- **如果类的内部有typedef,请把它放在类的起始处，因为防止先看到的是全局的和这个typedef相同的冲突，编译器会选择全局的，因为先看到全局的**

## 3.2：Data Member布局

Nonstatic data members在class object中的排列顺序将和其被声明的顺序一样，任何中间介入的static data members都不会被放进对象布局之中。

编译器还可能会合成一些内部使用的data members，以支持整个对象模型。vptr就是这样的东西，目前所有的编译器都把它安插在每一个“内含virtual function之class”的 object 内。一些编译器把vptr放在一个class object的最前端。

- **非静态成员变量的在内存中的顺序和其声明顺序是一致的**
- 但是**不一定是连续的**，因为中间可能有内存对齐的填补物
- virtual**机制的指针所放的位置和编译器有关**

## 3.3：Data Member的存取

### static Data Member

静态数据成员，按其字面意思，被编译器提出于class之外，并被视为一个global变量(但只在class生命周期内可见)。

每一个static data member只有一个实例，存放在程序的data segment之中。每次程序参阅（取用）static member时，就会被内部转化为对该唯一extern实例的直接参考操作。

从指令执行观点看，这是通过指针或对象存取member结论**完全相同的唯一一种情况**。这是因为通过对象存取操作只是文法上的便宜行事，因为静态成员不在类对象中，所以存取并不需要通过类对象。

如果是继承来的静态成员，依然是一样的，还是只有一个实例，存取路径仍然是那么直接。

如果是函数调用静态变量，没人知道会发生什么事情。

取一个静态成员变量的地址会得到指向其数据类型的指针，因为不在类对象之中。

两个类声明了相同的静态成员放在数据段不会名称冲突2，编译器暗中编码

### Nonstatic Data Members

非静态成员变量直接存放在类对象中，没办法直接调用，只能通过对象。

欲对一个nonstatic data member进行存取操作，编译器需要把class object的起始地址加上data member的偏移位置（offset）。指向data member的指针，其offset值总是被加上1，这样可以使编译系统区分出“一个指向data member的指针，用以指出class的第一个member”和“一个指向data member的指针，没有指出任何member”两种情况。

但是在继承体系下，情况就会不一样，因为编译器无法确定此时的指针指的具体是父类对象还是子类对象，所以这个存取操作被延迟到执行期。如果使用子类的类型声明便不会有这样的问题，即使数据继承自别的类

## 3.4：继承与Data Member

在C++继承模型中，一个derived class object所表现出来的东西，是其自己的members加上其base class（es） members的总和。至于derived class members和base class（es）members的排列顺序，则并未在C++Standard中强制指定；理论上编译器可以自由安排之。在大部分编译器上头，base class members总是先出现，但属于virtual base class的除外（一般而言，任何一条通则一旦碰上virtual base class就没辙了，这里亦不例外）。

- **只要继承不要多态**

一般而言，具体继承并不会增加空间或时间上的额外负担。

<img src="https://img-blog.csdnimg.cn/img_convert/3665e945f0501a2255977e55bf3f002f.png" alt="单一继承且无虚函数" style="zoom:50%;" />

这种情况下常见错误：

- 可能会重复设计一些操作相同的**函数**，我们可以把某些函数写成**inline**,这样就可以在子类中**调用**父类的某些函数来**实现简化**
- **把数据放在同一个类中和继承起来的内存布局可能不同，因为每个类需要内存对齐**

<img src="https://img-blog.csdnimg.cn/img_convert/09b5abfddaa566dcdc682dd037a8bef7.png" alt="分层继承的布局" style="zoom:50%;" />

- **容易出现的不易发现的问题：**

<img src="https://img-blog.csdnimg.cn/img_convert/fd37ad03de4e0313cad2d610f3753b0c.png" alt="继承下易犯错误" style="zoom:50%;" />

- 加上多态

  ### **当加上多态之后，对空间上增加的额外负担包括：**

  - **导入一个虚函数表，表中的个数是声明的虚函数的个数加上一个或两个slots(用来支持运行类型识别)**
  - **在每个对象中加入vptr，提供执行期的链接，使每一个类能找到相应的虚函数表**
  - **加强构造函数，使它能够为vptr设定初值，让它指向对应的虚函数表，这可能意味着在派生类和每一个基类的构造函数中，重新设定vptr的值**
  - **加强析构函数，使它能够消抹“指向类的相关虚函数表”的vptr,vptr很可能以及在子类析构函数中被设定为子类的虚表地址。**
    - **析构函数的调用顺序是反向的，从子类到父类**

​	vptr被放在最后，最前以及父子变量之间具体看编译器

- **多重继承**

  *单一继承特点：**派生类和父类对象都是从相同的地址开始，区别只是派生类比较大能容纳自己的非静态成员变量

​	多重继承下会比较复杂

<img src="https://img-blog.csdnimg.cn/img_convert/aaa62939adac39629627f21b1152eeb8.png" alt="多重继承关系" style="zoom: 67%;" />

- **一个派生对象，把它的地址指定给最左边的基类，和单一继承一样，因为起始地址是一样的，但是后面的需要更改，因为需要加上前面基类的大小，才能得到后面基类的地址**

<img src="https://img-blog.csdnimg.cn/img_convert/2085a8aac4e5a4b39d7dcd1cb3175af9.png" alt="多重继承数据分布" style="zoom:67%;" />

- 虚拟继承

![虚继承关系图](https://img-blog.csdnimg.cn/img_convert/ce27f9ab4a1f81ba64f84f87333600d0.png)

语言层面解决办法就是采用虚拟继承

```c++
class ios {...};
class istream : public virtual ios {...};
class ostream : public virtual ios {...};
class iostream :
	public istream, public ostream{...};

```

<img src="https://img-blog.csdnimg.cn/img_convert/61d413c54294ec0934c7cc5a39d0ccd6.png" alt="虚继承数据模型" style="zoom:67%;" />

一般而言，虚基类最有效的运用形式1，就是只有函数没有数据

## 3.5：对象成员的效率

- 程序员如果关心程序效率，应该实际**测试**，不要光凭推论、常识判断或假设。
- 优化操作并不一定总是能够有效运行，我不止一次以优化方式来 编译一个已通过编译的正常程序，却以失败收场

单一继承应该不会影响测试的效率，因为members被连续存储于derived class object中，并且其offset在编译时期就已知了。

虚拟继承的效率令人失望！两种编译器都没能够辨识出对“继承而来的data member pt1d::_x”的存取是通过一个非多态类对象（因而不需要执行期的间接存取）进行的。

## 3.6：指向Data Members的指针

如果vptr放在对象的头部，那么对成员取地址(通过类名取)(即取偏移量)，会比期望的多1byte。如果没多1，说明编译器处理过了

如何区分一个“没有指向任何data member”的指针，和一个指向“第一个data member”的指针？

为了区分p1和p2，每一个真正的member offset值都被加上1。不论编译器或使用者都必须记住，在真正使用该值以指出一个member之前，请先减掉1。

# 四、Function语意学

## 4.1：Member的各种调取方式

1. 非静态成员函数

   C++的设计准则之一就是：nonstatic member function至少必须和一般的nonmember function有相同的效率。所以编译器将member转化为了nomember

   转化步骤：

   - 改写函数原型，安插一个额外参数，this指针
   - 将操作改为经由this来存取
   - 重新写为一个外部函数

   名称特殊处理：在函数名前加上类名，如果有重载再加上参数链表
2. 虚拟成员函数

   ```c++
   ptr -> normalize();
   将会在内部被转化为
   (*ptr -> vptr[1])(ptr)
   ```

   - vptr表示由编译器产生的指针，指向virtual table。它被安插在每一个“声明有（或继承自）一个或多个 virtual functions”的class object中。事实上其名称也会被“mangled”，因为在一个复杂的class派生体系中，可能存在多个vptrs。
   - 1是virtual table slot的索引值，关联到 normalize()函数。
   - 第二个ptr表示this指针。
3. 静态成员函数 它**没有this指针**，因此会有以下**限制**：

   - **它不能直接存取类中的非成员变量**
   - **它不能够被声明为const、volatile 和 virtual**
   - **它不需要经过类的对象才能被调用----虽然很多事情况是这样调用的**

     如果取一个static member function的地址，获得的将是其在内存中的位置，也就是其地址。由于static member function没有this指针，所以其地址的类型并不是一个“指向class member function的指针”，而是一个“nonmember函数指针”。

## 4.2：虚拟成员函数

virtual function的一般实现模型：每一个class有一个virtual table，内含该class之中有作用的virtual function的地址，然后每个object有一个vptr，指向virtual table的所在。

- **多态含义：以一个基类的指针（或引用），寻址出一个子类对象**
- **什么是积极多态？**
- 当被指出的对象**真正使用**时，多态就变成积极的了

一个class只会有一个virtual table。每一个table内含其对应之class object中所有active virtual functions函数实例的地址。这些active virtual functions包括：

- 这一class所定义的函数实例。它会改写（overriding）一个可能存在的base class virtual function函数实例。
- 继承自base class的函数实例。这是在derived class决定不改写virtual function时才会出现的情况。
- 一个pure_virtual_called()函数实例，它既可以扮演pure virtual function的空间保卫者角色，也可以当做执行期异常处理函数（有时候会用到）。每一个virtual function都被指派一个固定的索引值，这个索引在整个继承体系中保持与特定的virtual function的关系。

<img src="https://img-blog.csdnimg.cn/img_convert/10faa6c278e3979ead5dbfb66e503942.png" alt="单一继承虚函数图" style="zoom: 67%;" />

```c++
Base2 *ptr = new Derived;
被转化为
Derived *temp = new Derived;
Base2 *ptr = temp ? temp + sizeof(Base1) : 0;
```

<img src="https://img-blog.csdnimg.cn/img_convert/837290da62d7c62f9ecdf3814aa6214b.png" alt="多继承虚函数图" style="zoom:67%;" />

<img src="https://img-blog.csdnimg.cn/img_convert/adf94e88a0957d2e969e2bc42c67b525.png" alt="虚继承下的虚函数图" style="zoom:80%;" />

## 4.3：函数的效能

nonmember、static member或nonstatic member函数都被转化为完全相同的形式。所以我们毫不惊讶地看到三者的效率完全相同。

## 4.4：指向member function的指针

- **对于普通的成员函数，编译器会将其转化为一个函数指针，然后使用成员函数的地址去初始化**
  - 例如 `double (Point :: *pmf ) ( );`
  - **转化为** `double ( Point :: coord )( ) = &Point :: x;`
  - **这样调用** `(coord) (& origin)或 (coord)(ptr)`
- **对一个虚函数取地址，在vc编译器下，要么得到vacll thunk地址（虚函数时候），要么得到的是函数地址（普通函数**

**指向虚拟成员函数的指针**

对一个非静态成员函数取地址，得到其再内存中的地址。然而对一个虚函数，其地址在编译期未知，所知道的仅是虚函数在虚表中的索引值。也就是说，对一个虚函数取地址，只能得到索引值。

**多重继承下，指向成员函数的指针**

为了让指向成员函数的指针能支持多重继承和虚拟继承，设计了一个结构体

```c++
struct _mptr{
    int delta;
    int index;
    union{
        ptrtofunc faddr;
        int       v_offset;
    };
};
```

delta 字段表示 this 指针的offset 值，而v_offset字段放的是一个virtual（或多重继承中的第二或后继的）base class的vptr位置。如果vptr被编译器放在class对象的起头处，这个字段就没有必要了，代价则是C对象兼容性降低。

index和faddr分别持有虚表索引和非虚拟成员函数的地址。

## 4.5：inline Function

一般而言，处理一个inline函数，有两个阶段：

1. 分析函数定义，以决定函数的“intrinsic inline ability”（本质的 inline能力）。“intrinsic”（本质的、固有的）一词在这里意指“与编译器相关”。如果函数因其复杂度，或因其建构问题，被判断不可成为inline，它会被转为一个static函数，并在“被编译模块”内产生对应的函数定义。
2. 真正的inline函数扩展操作是在调用的那一点上。这会带来参数的求值操作（evaluation）以及临时性对象的管理。

- 对于形式参数，会采用：
  - 常量表达式替换
  - 常量替换
  - 引入临时变量来避免多次求值操作
- 对于局部变量，会采用：
  - 使用临时变量
