# 1、STL概论与版本简介

## 01：stl概述

STL是可复用的标准模板库，在泛性思维上架设的一个概念结构，使抽象概念为主体，并使其系统化。

**六大组件**

1. 容器（containers）：各种数据结构，如vector、list、deque、set、map，用来存放数据。从实现来看，STL容器是一种 class template。就体积而言，这一部分很像冰山在海面下的比率。
2. 算法（algorithms）：各种常用算法如 sort、search、copy、erase，从实现的角度来看，STL算法是一种 function template。
3. 迭代器（iterators）：扮演容器与算法之间的胶合剂，是所谓的“泛型指针”，共有五种类型，以及其它衍生变化，从实现的角度来看，迭代器是一种将：operators*、Operator->、Operator++、Operator-- 等指针相关操作予以重载的class template。所有STL容器都附带有自己专属的迭代器——只有容器设计者才知道如何遍历自己的元素，原生指针（native pointer）也是一种迭代器。
4. 仿函数（functors）：行为类似函数，可作为算法的某种策略（policy），从实现的角度来看，仿函数是一种重载了 operator () 的 class 或 class template。一般函数指针可视为狭义的仿函数。
5. 配接器（adapters）：一种用来修饰容器（containers）或仿函数（functors）或迭代器（iterators）接口的东西，例如：STL提供的 queue 和 stack，虽然看似容器，其实只能算是一种容器配接器，因为 它们的底部完全借助 deque，所有操作由底层的 deque 供应。改变 functor 接口者，称为 function adapter；改变 container 接口者，称为 container adapter；改变 iterator 接口者，称为 iterator adapter。配接器的实现技术很难一言蔽之，必须逐一分析。
6. 配置器（allocators）：负责空间配置与管理，从实现的角度来看，配置器是一个实现了动态空间配置、空间管理、空间释放的 class template。

<img src="https://pic4.zhimg.com/v2-c7c0e6d0cb8a5a32101dbb9dc257a383_r.jpg" alt="preview" style="zoom: 33%;" />

## 02：STL文件分布与简介

- C++ 标准规范下的 C 头文件（无扩展名），如 cstdio、cstdlib、cstring
- C++ 标准程序库中不属于 STL 范畴者，如 stream、string
- STL 标准头文件（无扩展名），如 vector、deque、list、map、algorithm、functional
- C++ Standard 定案前，HP 所规范的STL 头文件，如 vector.h、deque.h、list.h、map.h、algo.h、function.h
- SGI STL 内部文件（STL 真正实现于此），如 stl_vector.h、stl_deque.h、stl_list.h、stl_map.h、stl_algo.h、stl_function.h

# 2、空间配置器

​	为什么不说allocator是内存配置器？因为空间不一定是内存，空间也可以是磁盘或者其他辅助存储介质。是的，可以写一个allocator直接向硬盘取空间。

​	SGI STL的配置器是alloc而不是allocator(未能符合标准规范，且效率不佳，只是对operator new和delete做了简单封装)

## SGI特殊的空间配置器

**注意std::alloc不接受任何模板型别参数**

### 01/构造和析构基本工具

<img src="https://pica.zhimg.com/v2-a2cf5fbb69b1a4783d6c65b47cfe43cc_720w.jpg?source=d16d100b" alt="preview" style="zoom: 67%;" />

- construct 接收 指针p 和 初值value，会将value设置到p所指的空间上。
- destroy 可以接收 指针、迭代器。
- - 基本类型指针：不做处理
  - 对象类型指针：调用析构函数
  - 迭代器：会判断析构函数是否为 trivial destructor（无用的、没必要的、无意义的析构函数）
  - - 是：则不做处理
    - 否：调用 迭代器中每个元素的析构函数。

这里有个问题是，如何判断是否为 trivial呢？

答案是：使用 __type_traits<T>::has_trivial_destructor() ，该函数会返回 __true_type 或 __false_type，前者代表是trivial，后者代表是有意义的。

### 02/空间配置与释放

<stl_alloc.h> 负责了对象构造前的空间分配和对象析构前的空间释放，有下面几个设计原则：

- 向 system heap 申请空间
- 考虑多线程状态（为了将问题简化，这里不讨论多线程状态）
- 考虑内存不足的应变措施
- 考虑过多小块内存造成的碎片问题

SGI 设计了双层分配器，如下图所示：

- 第一级直接使用 malloc 和 free
- 第二级，当需求大于128 bytes 时，调用一级分配器；小于等于128 bytes 则调用二级分配器。而且为了降低额外负担采用复杂的memory pool整理方式，不再求助第一级配置器

![preview](https://pic2.zhimg.com/v2-32301c3787030e27cdd97db42823f565_720w.jpg?source=d16d100b)

具体采用哪种分配器，需要看 __USE_MALLOC 是否被定义。

SGI 为 alloc 提供了一个 simple_alloc 的接口封装，使得外层使用时无需考虑内部具体用的一级还是二级。SGI STL 的容器都使用这个 simple_alloc 接口，而非直接使用 alloc。

<img src="https://pica.zhimg.com/v2-3861981acd62bef4ed9195158cea416a_720w.jpg?source=d16d100b" alt="preview" style="zoom:67%;" />

**（一）第一级配置器**

​	第一级配置器以malloc() free() remalloc()等C函数执行实际的内存配置、释放、重配置操作。并实现出类似C++ new-handler的机制(即内存配置不成功，改调用oom_malloc()和oom_remalloc()。后两者有内循环，不断调用，前提是被客端设定，否则直接报异常。)

​	若使用::operator new()来配置内存则失败后直接运用C++ new-handler调用客端设立的处理例程，这个有特定模式。alloc没有new-handler

(二)第二级配置器

二级分配器可以避免产生过多的小区块，可以解决内存碎片和过多的额外开销（系统需要多出来的空间管理内存，可以说是给系统“交税”）。

- 二级分配器以内存池（memory pool）管理小于128 bytes 的内存，称为次层分配（sub-allocation）：先分配一大块内存，组成一个自由链表（free-list），每次要取一定量内存时，从 free-list 中取；在用完后，分配器就归还给 free-list。
- 分配器会维护 空间为 8、16、24、……、128 这16个 free-list，在分配小内存时，会向上取整（Round Up），寻找最近的 free-list。
- free-list 节点结构是一个**联合体**，该节点在free-list中时，内容是一个指向 下一个节点的指针，在客户端使用时，是具体的数据。这样一物二用，不会造成维护链表指针的内存浪费。这个技巧在强类型语言（Strong Typed）中如 Java 行不通，但在弱类型语言（Weak Typed）中如 C++十分常见。

```c++
union obj{
    union obj * free_list_link;  
    char client_data[1];         // client use
}
```

<img src="https://pica.zhimg.com/v2-9bee0f1f628cf2ecbdf236425dc7c621_720w.jpg?source=d16d100b" alt="preview" style="zoom:67%;" />

填充过程：

1. 自己的不够了，找更大的区块看看
2. free list中无可用区块调用refill，为其填充空间
3. 新空间取自内存池(由chunk_alloc()完成)，缺省取地20新节点，不足就有多少取多少。
4. 如果内存池连一个都拿不出就用malloc从heap中取
5. heap都取不到，就去别的free list挖一块
6. 再没有，就调用malloc， malloc在调用oom_malloc处理机制
7. 最后只能报bad_alloc异常

## 内存基本处理工具

STL 定义了五个全局函数，除了前文提到的 construct 和 destroy，还有3个用来处理大块内存的复制和移动的 unitialized_copy、uninitialized_fill、uninitialized_fill_n 分别对应高层次的函数 copy、fill、fill_n。

unitialized_copy 函数让内存配置与对象构造行为分开。如果目标地址指向的空间都是未初始化区域，则会直接把源区域的对象产生复制品直接放到目标地址。STL 规范中要求该函数具有**原子性**，要么全部构造出来，要么全部不构造。

uninitialized_fill、uninitialized_fill_n 也和 unitialized_copy 类似。

这三个函数都会判断 对象是否为 POD（Plain Old Data，标量 or 传统 C 结构体），POD 会具有 trivial 函数，如果是 POD 则用最有效率的方法，如果非 POD 则用最安全的方法。过程大致如下所示。

<img src="https://pica.zhimg.com/v2-8a10ea90e4424730891fdc31e79afa3e_720w.jpg?source=d16d100b" alt="preview" style="zoom: 80%;" />

# 3、迭代器和traits

## 迭代器

《Design Patterns》中定义迭代器为：提供一种方法，使之能按序遍历某个容器的各个元素，又无需暴露该容器的内部表述方式。

在 STL 中，容器Container 和 算法Algorithm 是分离设计的，而 迭代器Iterator 就是两者的胶着剂。

- 迭代器是一种 smart pointer，而指针中最常用的两个操作是 解引用（dereference）和成员访问（member access），所以迭代器需要重载 operator * 和 operator ->。这个重载可以参考 C++标准库的 auto_ptr，它是用来包装原生指针（native pointer）的对象

如此一来可以为 list 设计一个迭代器，假设有如下 list 和 节点结构。

![img](https://pic2.zhimg.com/80/v2-aa58ebd755c77f35b189be6c80289425_720w.jpg?source=d16d100b)

根据之前所说的，类似指针的实现方式，当解引用时，返回 ListItem；当成员访问时，返回 ListItem* ；当递增时，得到下一个对象的 ListItem* 。可以实现迭代器如下。

![img](https://pic1.zhimg.com/80/v2-872458f076179da0d30a4040feaa077b_720w.jpg?source=d16d100b)

在 find 中是使用 *first != value 来检查元素是否匹配，但在这里， *first 是 ListItem<int>类型，value 是 int类型，两者并不能直接比较，所以还需要重载一个 operator != 使得 ListItem<int> 和 int 可以比较。

可以看到，为了针对 List 设计一个迭代器ListIter，暴露了很多 List 的内部实现细节。换句话说，设计迭代器，就需要对容器很熟悉。所以，实际上迭代器都是交由容器的设计者，将容器和迭代器的实现细节都封装起来，不对使用者可见。这也就是为什么 STL 容器都有自己专门的迭代器的原因。

## Traits

算法在使用迭代器时，可能会使用其关联类型（associated type），例如想声明一个 “迭代器所指对象的类型” 的临时变量。然而 C++ 只支持 sizeof()，不支持 typeof()。RTTI中的 typeid() 也不能拿来声明变量。这时候如何是好？

**（一）函数模板（function template）的参数推导（argument deducation）**

<img src="https://pica.zhimg.com/80/v2-a46861a045dabd6d5d6dcdb1b58ea197_720w.jpg?source=d16d100b" alt="img" style="zoom:80%;" />

这里以 func 为对外接口，实际操作交给 func_impl 完成，编译器对这个 function template 做了 argument deducation，于是得到了类型 T。

但这里是只能对**参数**做推导，如果想对返回值也做推导，则是不可能的。例如将 func_impl 的返回值从 void 改成 I，则编译器会报错。如何继续解决这个问题？声明内嵌型别似乎是个好主意。

**(二)内嵌类型**

这里可以在迭代器中声明一个内嵌类型，在其他函数中就使用该迭代器的内嵌类型。

<img src="https://pic3.zhimg.com/80/v2-e82c7bf768a3d0a237fa2e9fa6b90c5c_720w.jpg?source=d16d100b" alt="img" style="zoom:80%;" />

这里 typename 的作用就是在告诉编译器 I::value_type 是一个类型，而不是成员函数或数据成员。(必须加)

似乎已经解决了无法给返回值设定模板参数的问题，但对于原生指针呢？

前文提到，迭代器其实是一个指针，那显然 func 算法也需要接收原生指针作为参数，但原生指针是无法定义内嵌类型的。那么如何让上述方案对原生指针做特化处理，接收原生指针？

**(三)模板偏特化**

模板偏特化的意思是：如果 class template 有一个以上的 template 参数，我们可以对其中某个或多个但不为全部的参数进行特化处理。也就是在泛化设计中提供一个特化版本，将泛化版本中的某些参数赋予明确的指定。

​	假设有模板如下：

```c++
template<typename U, typename V, typename T>
class C { ... };
```

偏特化不是说一定要把 U、V、T 中的其中几个指定一个具体值，而是对它们做一定的约束。

> 《泛型思维》中定义其为：针对 template 参数做更进一步的条件限制所设计出来的一个特化版本。
>

于是就有了如下特化版本，仅仅接收原生指针

```c++
template<typename T>
class C<T*> { ... };
```

现在我们可以解决“内嵌型别”的问题：即在迭代器和容器之间再加一层**萃取机**

```c++
template<class I>
struct iterator_traits {
    typedef typename I::value_type value_type;
};

template<class T>
struct iterator_traits<T*> {    // T* 是原生指针，如 int*
    typedef T value_type;       // value_type 是 int
}
```

于是func改为：

> **模板传入了类型I，I分别传入func和萃取机(设置返回值)**
> **模板的模板？**
>
> ```c++
> template<class I>
> typename iterator_traits<I>::value_type
> func(I iter)
> { return *iter }   //模板传入了类型I，I分别传入func和萃取机(设置返回值)
>                    //模板的模板？
> ```
>
> 同理可以解决const T*类型，即再设计一个特化版本萃取
>

至此，iterator_traits::value_type 就能萃取出迭代器类型、原生指针类型、const 指针类型的 value_type 了。也就解决了 func 返回值类型的问题。

<img src="https://pic1.zhimg.com/80/v2-cb7f8c263bd461cc2b412312cc39bbf7_720w.jpg?source=d16d100b" alt="img" style="zoom:67%;" />

如上图所示，traits 就是在萃取各个迭代器的特性，这里的特性，指的就是关联类型（associated types）。想让 traits 正常运作，自定义的迭代器就必须得自行定义出各个关联类型的内嵌类型（nested type），如果没有定义则不能与 STL 兼容。根据经验，最常用到的类型有下面五种。

- value_type：迭代器所指对象的类型
- difference_type：两个迭代器中间的距离，也可以表示容器的最大容量
- reference_type：迭代器所指对象的引用（左值）
- pointer：迭代器所指之物的地址
- iterator_category：迭代器类型，见附录（二）

<img src="https://pic1.zhimg.com/80/v2-033d69f6c37118f19ab4f120261ff6f6_720w.jpg?source=d16d100b" alt="img" style="zoom:67%;" />

## 迭代器类型

执行期间才决定使用哪一个版本影响程序效率(使用if else判断)，重载机制可以解决这个问题，又可以利用萃取机，得到需要的迭代器类型。

按说advanced()可以接受各种类型的迭代器，就不该将泛化版本命名为输入迭代器(命名规则决定必须这样，以算法能接受的最低阶迭代器类型命名)，

于是引入了继承。

- iterator_traits负责萃取迭代器的特性
- _type_traits负责萃取型别特性，将是否为系统默认析构之类的，返回_false_type和_true_type
- #

# 4、序列式容器

![image-20220401221237268](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220401221237268.png)

​                                       	基层与衍生层的关系

​	所谓序列式容器，其中的元素都可序(ordered)，但未必有序(sorted)

## vector

- 迭代器

  vector维护的是一个连续线性空间，所以无论其元素类型为何，普通指针都可以满足所有必要条件，如：重载的一些运算符，普通指针天生就具备，vector支持随机存取，普通指针有这样的能力，所以提供的是Random Access Iterators（普通指针）
- 数据结构：连续线性空间start finish end_of_storage
- 构造与内存管理

  vector提供许多的构造器，其中一个允许指定空间大小和初值

  uninitialized_fill_n根据第一参数型别，决定用fill_n或反复调用构造函数

  vector的空间配置器是data_allocator，也就是simple_alloc，simple_alloc的实现就是std::alloc，根据申请的内存大小，决定用第一级配置器（malloc、free）还是第二级配置器（内存池），所以vector应该是分配在堆上的。

​	**事实上，扩容的时候，是这样一种机制**：会去别的地寻找一块空闲的内存，然后把原来的东西搬过去，这样子所谓的扩充，而不能在原地扩充。

​	拷贝用的是uninitialized_copy，如果拷贝的是POD（标量型别，也就是trivial）调用的是copy（自己去看STL 的copy实现），如果是non-POD使用for循环遍历，调用construct，一个一个的构造，针对char*和wchar_t*，uninitialized_copy直接用memmove来执行复制行为，更加快。

- 插入操作

  1：判断插入元素数量n是否为0：只有不为0才能执行所有操作

  2：判断备用空间是否大于“新增元素个数”。

  ​		uninitialized_copy：从前往后复制
  ​        copy_backward：从后往前复制

  ​		1：大于等于：开始判断“插入点之后元素个数”大于“新增元素个数”

  ​					1：大于的话：先uninitialized_copy后边多出来的元素到后边，然后finish后移，然后copy_backward剩余的,最后						fill插入

  ​					2:小于等于：先填充初值到新值大小，并将finish移到此处，然后uninitialized_copy旧值到新位置，并移动finish到						结尾，最后fill新元素

  3：备用空间不够，扩容到旧元素加新元素个数

![image-20220401231446852](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220401231446852.png)

![image-20220401231507347](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220401231507347.png)

## list

list每插入或删除一个元素，就配置或删除一个元素空间。而且list对任何位置的插入删除都是常数时间

- 节点：双向链表，不过指针类型是void*，可以设为_list_node<T>*
- 迭代器Bidirectional iterators。list有一个重要性质，插入和接合都不会使迭代器失效。迭代器是结构体类型，但内含普通指针
- 数据结构：SGI list不仅是双向链表还是环状双向链表，尾端是空白节点

## deque

- 头尾增删，没有容量概念
- 中控器![image-20220402122226303](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402122226303.png)![image-20220402122300715](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402122300715.png)![image-20220402165622787](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402165622787.png)
- 数据结构：start、finish指向第一个缓冲器第一个和最后一个缓冲区尾部![preview](https://pic2.zhimg.com/v2-0523596236d348211058f5ba7f6cdfcd_r.jpg)
- 构造与内存管理：两个专属空间配置器：一个配指针大小，一个配元素大小

## stack

以deque或list形成，只需封闭开头

## queue

![image-20220402174444833](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402174444833.png)

同样无迭代器、不允许遍历

## heap

不是容器，是幕后英雄。底层是完全二叉树

![image-20220402175024138](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402175024138.png)

![image-20220402175042399](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402175042399.png)

push_heap先放到底部，再调用上溯，换位置，直到不需兑换或直到根节点

pop_heap取的是根节点

## priority_queue

缺省情况下，是一个max-heap，是一个以vector表现得完全二叉树

![image-20220402182552481](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220402182552481.png)

**依照权值自动递减** pop即vector的pop_back()

## slist

单向迭代器

# 5、关联式容器

## map

​	map特性，所有元素根据键值自动排序，键值不允许重复，数据类型为pair，元素的属性是public。迭代器无法修改键值，会影响排序。当删除元素时，不影响别的迭代器。

​	map使用红黑树的insert_unique, multimap使用insert_equal.

​	使用find应该使用专属的，更有效率。

​	插入操作的返回值类型是pair，由一个迭代器和bool值组成。

​	下标操作符，可以作为左值或右值(内容不可修改)，可以这样是因为，返回值是引用类型

![image-20220405121255194](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220405121255194.png)

过程分析：首先根据键值和实值产生一个元素，由于实值未知，所以产生一个临时对象代替，然后在将该元素插入到map里。

如果是左值引用，恰好我们添加新元素，将位置卡好；如果是右值引用，我们取其实值

- multi版本除了插入用insert_equal以外都一样

## hashtable

![image-20220405122316868](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220405122316868.png)

**线性探测**：使用这种方法时，如果我们想要将一个元组( k , v ) (k,v)(*k*,*v*)插入桶A [ j ] A[j]*A*[*j*]处（这里j = h ( k ) j=h(k)*j*=*h*(*k*)），但是A [ j ] A[j]*A*[*j*]已经被占用，那么我们将尝试尝试插入A [ ( j + 1 )  m o d  N ] A[(j+1)\ mod\ N]*A*[(*j*+1) *m**o**d* *N*]，若A [ ( j + 1 )  m o d  N ] A[(j+1)\ mod\ N]*A*[(*j*+1) *m**o**d* *N*]也已经被占用，则我们尝试使用A [ ( j + 2 )  m o d  N ] A[(j+2)\ mod\ N]*A*[(*j*+2) *m**o**d* *N*]，如此重复操作，直到找到一个可以接受新元组的空桶。

平均插入成本的成长幅度，远高于负载系数的成长幅度，这种现象称为主集团。

虽然线性探测可以节省空间，但是它存在的问题是：线性探测倾向于将一个映射的元素集中地存储，这种使用连续的哈希单元的运行方式会导致搜索速度大大降低。

**二次探测：**它反复探测桶A [ ( h ( k ) + f ( i ) )  m o d  N ] ,  i = 0 , 1 , 2 , . . . A[(h(k)+f(i))\ mod\ N],\ i=0,1,2,...*A*[(*h*(*k*)+*f*(*i*)) *m**o**d* *N*], *i*=0,1,2,...，其中f ( i ) = i 2 f(i)=i^2*f*(*i*)=*i*2，直到发现一个空桶。与线性探测一样，二次探测会使删除操作更复杂，但它确实可以避免在线性探测中发生的聚集模式。

**迭代探测**：另一种避免聚集的开放寻址方法是迭代地探测桶A [ ( h ( k ) + f ( i ) )  m o d  N ] A[(h(k)+f(i))\ mod\ N]*A*[(*h*(*k*)+*f*(*i*)) *m**o**d* *N*]，这里f ( i ) f(i)*f*(*i*)是一个基于伪随机数产生器的函数，它提供一个基于原始哈希码位的可重复的但是随机的、连续的地址探测序列。python的字典类现在就是使用这种方法。

**开链**

![image-20220405171155289](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220405171155289.png)

hashtable的迭代器_hash_table:不能后退

![image-20220405173815144](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220405173815144.png)

数据结构：buckets聚合体以vector完成

![image-20220405181449529](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220405181449529.png)

构造与内存管理

![image-20220405181536089](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220405181536089.png)

[【STL源码剖析】总结笔记（10）：哈希表（hashtable）探究 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/439650164)

# 6、算法

## 算法泛化过程

以find算法为例：

一、使用array，传入指针，容器大小，和要寻找的值。缺点太依赖于容器，且暴露了很多细节

二、改进：输入两个int指针参数，标出一个区间。缺点：依赖书据类型

三、为了摆脱数据类型使用模板，传值使用pass by refference

四、为了摆脱只能使用指针，使用迭代器。

## 数值算法

**accumulate**

![image-20220406183656463](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406183656463.png)

**adjacent_difference**

默认得到两相邻元素的差值，有仿函数则仿函数处理得到结果

**inner_product**

![image-20220406184224435](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406184224435.png)

![image-20220406184246276](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406184246276.png)

**partial_sum**

默认处理前n-1个数的结果，与第n个数做仿函数处理

## 基本算法

equal：容器1的头尾。容器2的头，仿函数

![image-20220406190356010](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406190356010.png)

fill

![image-20220406190518462](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406190518462.png)

iter_swap

![image-20220406190758334](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406190758334.png)

![image-20220406190816990](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406190816990.png)

max和min

![image-20220406191440106](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406191440106.png)

mismatch

![image-20220406191546686](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406191546686.png)

### copy

![image-20220406191653934](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406191653934.png)

注意：第二个容器的起点，不能在第一个容器的范围内，如果两个容器相同的话

## set相关算法

set_union构造并集，返回迭代器指向区间的尾端，且从小到大排好序了

set_intersection交集

set_difference差集

set_symmetric_difference对称差集，上边是一个，下边是两个的并集

## 其他算法

adjacent_find找出第一组相邻的相等元素，返回的是值

![image-20220406195041426](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406195041426.png)

![image-20220406195145948](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406195145948.png)

![image-20220406195203725](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406195203725.png)

![image-20220406195444998](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220406195444998.png)

# 7、仿函数

## 仿函数概观

![image-20220407101448095](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407101448095.png)

## 可配接的关键

为了拥有配接能力，每一个仿函数必须需定义自己的相应型别，就像迭代器i想要加入STL大家庭，也必须依照规定定义自己的五个相应型别一样。这些相应型别是为了让配接器能够取出获得仿函数的某些信息，相应型别都是只是一些typedef，所有操作在编译期间就完成了，不带来任何额外负担。

![image-20220407103004945](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407103004945.png)

![image-20220407103208901](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407103208901.png)

## 算数仿函数

```

```

![image-20220407104028550](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407104028550.png)

## 关系运算符

![image-20220407104247135](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407104247135.png)

## 逻辑运算类

![image-20220407104324539](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407104324539.png)

# 8、配接器

## 配接器分类

​	stl提供的各种配接器中，改变仿函数接口者，我们称为function adapter。改变容器接口者，称为container adapter。改变迭代器接口者，称为iterator adapter。

- 应用于容器queue和stack
- 应用于迭代器![image-20220407110558703](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407110558703.png)![image-20220407110625161](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407110625161.png)
- 应用于仿函数

  ![image-20220407182638392](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407182638392.png)![image-20220407182704653](D:\学习课件\C++\读书笔记\STL源码剖析.assets\image-20220407182704653.png)
