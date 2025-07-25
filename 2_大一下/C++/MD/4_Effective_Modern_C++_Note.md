+++

[toc]

+++

# `Effective Modern C++`

## 关于类

### 初始化

`C++`丰富的语法中提供了三种对象的初始化：

```c++
int a = 0;
int a(0);
int a{ 0 };
```

一般来说，对数组的初始化通常只用花括号，忽略等号

这样的初始化会带来对类类型的疑惑，这里复习一下：

```c++
Class a = ClassObj;	// 调用复制构造函数.
a = ClassObj;		// 调用重载赋值运算符.
```

`C++`通过花括号来实现统一初始化，只有花括号能在任何地点使用，既不会被看作函数声明，也可用在不可拷贝类上，你也可以自定义初始化列表构造函数，编译器会优先调用它而不是其它构造函数，只有在这种情况下，可以考虑使用圆括号调用其它构造函数，但如果需要自定义，一定需要三思，因为用户可能因这个自定义而疑惑

一种边缘情况是，在一个类既有默认`ctor`又有自定义初始化列表`ctor`的时候，传递空花括号时编译器会选择默认`ctor`

使用花括号唯一的限制是不能对内部表达式进行变窄转换(而其它两种因向下兼容问题不会检查这种转换)

总而言之，无论是作为库开发者还是库使用者，应该注意是否含有初始化列表`ctor`，事实上`()`与`{}`各有优劣，但后者使用更广

### 成员函数的引用限定符

在`C++11`中，增加了两种限制调用成员函数的情况，即调用左/右值的成员函数，与`const`一样，可以作为签名区分两个重名函数：

```c++
class A {
public:	// 这两个限定符要放在const后.
    void member() const &;	// 左值限定符,只有左值能够调用它.
    void member() const &&;	// 右值限定符,只有右值能够调用它.
};
```

### 对于容器优先使用常量迭代器和非成员迭代器函数

优先考虑常量迭代器不难理解，优先使用非成员迭代器函数的原因是：

- 原生`C`数组不能使用成员函数，可能还有更多情况
- 自由函数拥有更大的通用性，因为可以随时添加对某个特定容器的特例化版本，而不用对这个数据结构进行改动

### 构造函数、析构函数、虚函数、赋值运算符

- 多态性决定了，可以通过用基类指针来使用子类方法，对于这种基类，应将析构函数声明为虚拟的；因为使用的是子类的数据，就应该使用子类的析构函数

  ```c++
  class Base {
  public:
      virtual ~Base();
  };
  class Son : public Base {
  public:
      ~Son();
  };
  int main() {
      Base* Obj = new Son;	// 应该调用Son的析构函数.
  }
  ```

- 不在构造、析构函数中使用虚函数，原因很简单，编译器会先构造父类，再构造子类，而一旦在构造父类对象时调用虚函数，那么就会调用一个尚未构造的子类对象的成员函数，这是危险的；析构函数同理，先析构子类对象再析构父类对象，在调用虚函数时子类对象早已销毁了

- 永不从析构函数中抛出异常，就算需要抛出异常，也应由析构函数捕获并终止程序

- 在实现使用堆内存类的赋值运算符重载时，需要考虑自我赋值的情况，一般不能先释放自身内存再拷贝右值资源：

  ```c++
  Class& operator=(const Class& rhs) {
      // delete data;	// 错误!万一rhs是自己呢?
      
      if (rhs == this) {return *this; }	// 正确做法一:检查,但会使代码量增加.
      
      Class* tmp {data};	// 正确做法二:先记住原先data,不会增加代码,但有暂时空间开销.
      data = new type(rhs.data);	// 复制,这里已经创建新副本.
      delete tmp;		// 删除原先data.
      return *this;
  }

### 保证常成员函数的线程安全

线程安全是指，需要在并发环境下，保证获取、改变资源的安全：

- 一般来说，同时调用多个**只有读操作**的常成员函数不会造成线程安全问题
- 当多线程同时工作时，如果含有写线程，则可能引起**数据竞争**，即有不同线程同时修改共享数据，这破坏了线程安全，例如：
- 多窗口同时售票情景，需要保证已取走的共享资源不会被其它线程访问
- `C++11`引入`mutable`，它只能修饰成员变量，表示数据一直可变(**即使在`const`成员函数中**)。例如，它可用于缓存已计算的值，这使得`const`成员函数只需要计算第一次，而在之后都访问这个结果。由于它能在常成员函数中被改变，所以即使是`const`成员函数，也引发了线程安全问题

有以下几种方式保证线程安全：

- 使用`std::mutex`，即互斥量：

  ```c++
  #include <mutex>
  class A {
  public:
      void Func() const {
          lock_guard<mutex> lg(m);	// 锁住互斥量m,使其它线程无法访问.
          if (!Value) {	// 如果没有记录.
              Compute;	// 计算并记录.
          }
          return Value;	// 返回结果.
      }	// 解锁m.
  private:
      mutable mutex m;	// 由于lock_guard调用非常成员函数,所以要声明multable.
      mutable Type Value;
  };
  ```

- 使用互斥量开销较大，可使用`std::atomic`：

  ```c++
  #include <atomic>
  class A {
  private:
      mutable atomic<unsigned> cnt;	// 只是用于记录调用次数.
  };

当然，如果能保证一个常成员函数不会发生在并发环境或者不会发生写操作，那么它的线程安全是无关紧要的

### 为类型信息添加类型萃取类`type_traits`

假设需要统一不同类的接口，而这些类的一些类型在实现上有区别，最好使用类型萃取来管理这些类型信息

例如，`C++ STL`的迭代器类型就依靠这种技术：

- 链表与数组的迭代器是不同的，前者是一个具体类，而后者只是原生指针
- 使用部分模板特化，对类型信息进行萃取，并使算法端通过这个萃取类来访问迭代器
- 萃取类尝试获取容器的迭代器类，如果没有，则返回原生指针
- 这样就分开了算法与容器，使得算法不用考虑迭代器的实现，而总是获取到想要的迭代器

使不同的类型包括自定义类、基础数据类型等有统一的类型名，实现统一的接口，这就是类型萃取的任务

## 智能指针

使用智能指针需要包含头文件`<memory>`

### `std::unique_ptr`管理独占资源

在以前的`C++`版本中，智能指针由`std::auto_ptr`担任，但`auto_ptr`是在`C++11`之后被废弃的，它有如下问题：

- 智能指针是类，设计原意为拥有原生指针的功能，并且能避开原生指针的坑，例如使用的是栈空间还是堆空间？如果使用堆空间，那么它是单个对象还是数组(决定使用`delete`还是`delete[]`)？
- 用拷贝方法来实现资源转移，但为了保证资源所有权唯一，**原本的副本会被赋值`nullptr`**，对于其它指向这块副本的智能指针来说是危险的；在语义上，给人"拷贝"的错觉，但其实并非拷贝
- 仍然需要手动`delete`

而`unique_ptr`拥有如下优势：

- 用移动方法来实现资源转移，不允许拷贝，这在语义上是很清晰的
- 与智能指针的初衷一样，每个`unique_ptr`拥有唯一的资源(除非它是`nullptr`)，每份资源对应唯一的`unique_ptr`，不会导致多次`delete`
- 非`nullptr`指针析构时自动释放指向的资源，能安全地`delete`

所以对于独占资源，最好使用`unique_ptr`来管理，如果使用原生指针，则很有可能因多次`delete`而出错

### `std::shared_ptr`管理共享资源

通过`shared_ptr`，`C++`拥有一套自动垃圾回收，且可预测回收时机的系统。它使用的垃圾回收策略是**引用计数**，事实上很多语言并不使用这套策略，因为它有一些问题：

- 只有在最后一个引用对象析构时，`shared_ptr`会`delete`掉这部分资源，但对循环引用的资源无效
- 性能上的问题：它需要同时维护一个引用计数，而且这个引用计数是动态分配的
- 每个`shared_ptr`只对应一个控制块，即一份资源，但每个`shared_ptr`控制块互相不关联，这导致下面这个问题：
  - 通过原始指针创建一个`shared_ptr`，将同时**创建一个控制块**，接下来正常应该只通过它来管理，但如果再次用**同一原始指针**创建新的`shared_ptr`，**尽管它们指向的内存相同**，它依旧会创建一个**新的控制块**，这将导致重复`delete`
- 用`shared_ptr`管理的资源应该在堆上，因为它总想使用`delete`来释放这份资源

尽量养成以下习惯：

- 使用`make_shared`来创建控制块而不是传递一个原始指针，它保证在堆上创建新对象，而这个新对象肯定不会与其它控制块关联
- 通过传递`shared_ptr`或`weak_ptr`来创建新引用，这样不会创建新的控制块
- 由于有额外开销，尽量只在共享资源上使用它
- 使用`weak_ptr`辅助可能悬空的`shared_ptr`
  - ``weak_ptr`是`shared_ptr`的升级，当它悬空时，`expired`函数返回`1`，但是，它无法解引用，通常来说，`weak_ptr`用于观测一份资源，或用在循环引用中打破循环的引用计数
- 不混用智能指针和原始指针

### 使用`make`函数而不是`new`

常用的`make`函数有`make_unique`(`C++14`)和`make_shared`(`C++11`)，这有以下好处：

- 两种方式的区别：

  ```c++
  #include <memory>
  auto upObj{std::make_unique<Class>()};	// 调用函数申请一份Class资源.
  std::unique_ptr<Class> upObj {new Class};
  ```

- 光是这样看似乎没有区别，但后者隐含了**异常安全**问题，因为在调用`new`和调用`unique_ptr`构造函数间有一层空隙，如果在这时出现异常，那么已`new`的资源无法被`unique_ptr`管理，造成**内存泄漏**；而前者安全性更高

- 但是，`make`函数不支持自定义删除器和花括号初始化，在这些情况下`make`函数不好用

## 优先考虑的关键字与场景

### 优先使用`nullptr`而不是`0`或`NULL`

- `NULL`是`0LL`的宏定义，与`0`都被看作数值，只有在必须转换成指针时才会转换，这样的优先度使得当遇到重载的数值参数函数和指针参数函数时，调用`func(NULL)`可能不是你想要的
- `nullptr`增加了类型安全的保险，它只能隐式转换成任意的指针类型，编译器不会将它视为数值
- `nullptr`是向后兼容的，可以完美替换掉老代码中的`NULL`
- `nullptr`可以直接使用，那么应该是某种类型的实例化，标准定义为`decltype(nullptr)=std::nullptr_t`

### 优先使用`using`而不是`typedef`进行重命名

- `typedef TypeName OtherName`与`using OtherName = TypeName`都是将`TypeName`重命名为`OtherName`，某些情况下`using`可读性更高，例如：

  ```c++
  void func(int a);
  typedef void (*OtherName)func(int);	// 将这个函数指针类型重命名为OtherName.
  using OtherName = void (*)func(int);// 效果相同.
  ```

- 在模板类型的重命名上，`using`更简洁实用，例如：

  ```c++
  template <typename T>
  class basic_List<T, Alloc<T>> {};
  // 为了让用户声明List<T>时就像声明basic_List<T, Alloc<T>>时一样.
  template <typename T>	// typedef是做不到的,最多只能嵌套在结构体内.
  struct ListType {
      typedef basic_List<T, Alloc<T>> type;
  };
  // 并且由于这个类依赖于T, 在其它模板类使用时需要在前面加typename让编译器将它视为类型.
  template <typename T>
  class UseList {
  	typename ListType<T>::type _M_List;
  };
  
  // 而using则简单很多.
  template <typename T>
  using List = basic_List<T, Alloc<T>>;
  template <typename T>
  class UseList {
  	List<T> _M_List;	// 正常使用,达成目的.
  };

### 优先使用限域`enum`而不是非限域

非限域`enum`指，一个枚举类型的所有内含名在这个`enum`所在的作用域都不可再使用，如果没有限定在命名空间里或限域，这些内含名就会泄露到整个文件中

限域指`C++11`的一个新特性：

```c++
enum class ENUM { A, B, C };// 在枚举类型前加class,将枚举名限定在花括号内,且总可以前置声明.
auto a = ENUM::A;	// 通过枚举类型访问枚举名.
```

### [优先考虑`auto`而不是显式声明类型](#auto)

### 优先使用`delete`而不是未定义的私有函数

当不希望用户使用一个类的构造函数，而编译器又会偷偷提供时，以前的做法是在私有域内声明，且不定义它们，但这样会使友元或成员函数有潜在的使用风险，道理很简单，应该使用`delete`关键字：

```c++
class A {
public:
    A(const A&) = delete;
};
```

`delete`不止可用于成员函数，它能用于任何函数以处理隐式转换的弊端：

```c++
void func(int);		// 如果希望只接受int,而不是char,bool,double,就删除它们.
void func(char) = delete;
void func(bool) = delete;
void func(double) = delete;	// float会先转换成double,所以这条同时删除了float.
```

它也能用于删除特化模板的实例化

### 优先使用`noexcept`只要函数不抛出异常

`C++11`引入的`noexcept`在以前是`throw()`，但拥有更多的优化灵活

对于一个绝对不会抛出异常的函数来说，可以在后面添加`noexcept`来承诺，允许编译器**极尽所能优化**生成的代码

而且，`noexcept`可视条件优化，即：

```c++
template <typename T>
void swap(T& a, T& b) noexcept(noexcept(swap(*a, *b)));
```

即只要提供不抛异常的`swap(*,*)`，那么`swap(&,&)`就是不抛异常的

使用`except`需要考虑改动，因为只要承诺了，就要考虑用户的使用，未来一旦改动，那么向下的代码都有可能出现问题，一般而言，**移动、交换、内存释放**等相关函数能靠`noexcept`得到相当大的优化

### 尽可能使用`constexpr`

`constexpr`是`C++11`引入的关键字，从文字上来看，是`const`与`expr`的组合，即常量表达式：

- 对于`constexpr`对象而言，与`const`对象不同的是它需要在编译期就决定，即编译期常量
- 对于`constexpr`函数而言，如果传递`constexpr`对象，函数会返回一个编译期常量，否则将在运行时计算；注意这不表示函数返回一个`const`值，而是"如果传递编译期常量参数，则返回值可作为编译期常量"
- 注意在`C++11`中，只能用`1`条`return`语句，而在`C++14`中限制减小了
- 注意`constexpr`与`const`、`noexcept`同为接口的一部分，如果未来可能修改，那么最好不要使用它们

## 类型推导细则

### 模板类型推导

通常通过定义模板类或模板函数来减少代码量，使用模板所要求的是，每个使用过它的`.cpp`都能直接看见它的定义，即声明与定义应该放在同一份头文件内

因为不同参数类型的模板作为头文件在展开时需要实例化，而展开过程在预处理期，所以当声明与定义分开时，将因头文件找不到定义而展开失败，导致无法编译

而一般头文件的展开只是复制粘贴而已，不影响编译，那么整个过程就不会出错

放在一起时编译器会丢弃重复的实例化，最后随机保留一份；事实上`inline`、`module`等新特性正在致力于将所有定义放进头文件内或直接不使用头文件

通常来说，对于一个模板，编译器会对`T`与`ParamType`推导类型：

```c++
template <typename T>
void func(ParamType Param);
```

`ParamType`可以有一些修饰符，但终究`ParamT`会被推导为非引用非指针类型，例如：

```c++
template <typename T>
void func(const T& Param);
```

这里函数签名内的`T`会推导成模板参数`T`，这是最简单的情形：当传入的`ParamType`是一个指针或一个非通用引用时(只能作为左值或右值之一的引用)，`T`就会是非引用非指针类型

如果`ParamType`为通用引用(既可能是左值，也可能是右值)，在作为右值时当然与上述情况相符，但作为左值时，考虑以下模板：

```c++
template <typename T>
void func(T&& Param);
int x = 1;
func(x);	// 此时Param为左值,推导出不正常的T = int&.
func(1);	// 此时Param为右值,推导出T为正常的T = int, ParamType为int&&.
```

只有在引入了右值引用的`C++11`中才出现通用引用，在类型推导时会区分引用的左值与右值

### `auto`类型推导<a id="auto"></a>

一般进行`auto`的情况有例如申请迭代器等场合，事实上除了推导花括号的初始化外，其它情况均与模板推导相同，而对于内部元素类型相同的花括号，`auto`能够成功推出它是一个初始化列表，以及它的模板参数

`auto`声明变量一定要求初始化，这在很多情况下比不用`auto`要**安全**；除此之外，使用它也可以解决某些**移植性问题**

`auto`也可用于函数返回类型的自动推导，但单独使用`auto`推导返回类型比较少用，由于与模板推导是及其类似的，推出的类型大部分情况下是**非引用非指针**类型的，这在很多情况下会造成麻烦

使用`auto`时需注意一些特殊情况，例如隐藏的代理类调用：

例如对一个`vector`容器，它在实现`bool`时**采取`1`位存储**，由于`C++`不允许对位的引用，导致`vector<bool>`的`operator[]`只能返回一个代理类`vector<bool>::reference`对象来模仿引用的行为，这时使用`auto`推导出的对象是实实在在会改变容器的(而不像其它类型返回的是值)

在上述情况下，可以考虑使用显式类型转换器，这能保证既进行了初始化，又进行了类型转换：

```c++
vector<bool> vec;
auto value = static_cast<bool>(vec[0]);	// 结果为bool.
auto reference = vec[0];	// 结果为vector<bool>::reference.
```

当然，一般的`bool`空间是八位，一个字节，可以用`bool&`

### `decltype`

`decltype`是`C++11`新增的关键字，用于在编译期推导出表达式内的类型，并用这个类型作为后面变量的类型

它与`auto`强制要求初始化不同，它只是帮你推导出你提供的表达式最终的类型而已，而且它与模板类型推导规则是不同的，对非直接变量名进行`decltype`，通常**返回它的引用**

`decltype`通常与尾置返回类型相匹配使用，尾置返回类型这个`C++11`引入的新特性在声明复杂函数时有奇效：

```c++
// 定义返回一个一维数组类型的函数.
int (* func(int arr[][3], int n) )[3];	// 一般声明.
auto func(int arr[][3], int n) -> int(*)[3];	// 尾置返回类型声明.
```

这里的`auto`不代表让编译器自动推导类型，而是告诉编译器使用尾置类型

但当函数复杂度极高时，例如定义一个返回上述`func()`的指针的函数时：

```c++
auto ptf() -> int (*func(int[][3],int))[3];
auto ptf() -> decltype(func);
```

这并没有快捷多少，所以`decltype`就派上用场了

还有一个场景，设想一个`operator[]`函数，我们需要它返回元素的引用：

- `auto`会去除引用
- `decltype`将推导出引用类型

那么组合起来，就像下面这样：

```c++
auto func(int arr[], int n) -> decltype(arr[n]);
// 或:
decltype(auto) func(int arr[], int n);
```

`decltype(auto)`是指，让编译器自动推导类型，并且使用`decltype`的规则

将数组推广为容器，那么通常传的是它的引用，这意味着只能传左值引用，事实上，虽然传入临时对象返回它的元素的引用是未定义的，但极限情况是有用的，如果需要支持它，可以使用通用引用：

```c++
decltype(auto) func(Container&& c, Index i);
```

