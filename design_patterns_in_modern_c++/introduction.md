# Introduction

设计模式这个话题听起来可能枯燥、学术性过强，而且老实说，在几乎可以想象到的每种编程语言中都已经讨论得相当透彻——包括像 JavaScript 这样并非完全面向对象的编程语言！那么，为什么还需要另一本关于它的书呢？

我认为这本书存在的主要原因是 C++ 又重新变得强大起来。在经历了长时间的停滞之后，它现在正在进化和发展，尽管它必须与向后兼容 C语言的问题作斗争，但好的变化确实在发生，只不过不是我们所有人都希望的速度。（我这里指的是模块化等特性。）

接下来谈谈设计模式——我们不应忘记，原始的设计模式书籍《设计模式：可复用面向对象软件的基础》是以 C++ 和 Smalltalk 为例出版的。自那以后，许多编程语言已经直接将设计模式融入了语言本身：例如，C# 通过内置对事件的支持（以及相应的 `event` 关键字）直接实现了观察者模式。C++ 并没有这样做，至少在语法层面上是这样的。不过，`std::function` 等类型的引入确实使很多编程场景变得更加简单。

设计模式也是对一个问题如何能够以多种不同方式解决的一种有趣探讨，这些方式在技术复杂度上各有差异，并涉及不同类型的利益权衡。一些模式或多或少是必不可少且不可避免的，而其他模式则更像是学术上的好奇心（但仍然会在本书中讨论，因为我是一个完美主义者）。

读者应当意识到，对于某些问题提供全面解决方案（例如观察者模式）通常会导致过度设计，即创建比大多数典型场景所需更为复杂的结构。虽然过度设计很有趣（嘿，你可以真正解决问题并给同事留下深刻印象），但这往往是不可行的。

## Preliminaries

### Who This Book Is For

本书旨在成为经典GoF（Gang of Four，四人组）设计模式书籍的现代更新版，特别针对C++编程语言。我想问一下，在座有多少人还在编写 Smalltalk？恐怕不多，这是我的猜测。

本书的目标是探讨如何应用现代 C++（即当前可用的最新版本的 C++）来实现经典的设计模式。同时，本书也尝试提炼出一些对C++开发者有用的新的模式和方法。最后，在某些章节中，本书实际上是现代C++的技术演示，展示了其最新特性（例如协程）如何使一些困难的问题变得更容易解决。

### On Code Examples

本书中的所有代码示例都适合投入生产环境，但为了提高可读性，做出了一些简化：

- 使用 `struct` 而非 `class`：很多时候，我会选择使用 `struct` 而不是 `class` 来避免在多个地方重复写public关键字。
- 省略 `std::` 前缀：为了避免影响可读性，特别是在代码密度较高的地方，我将省略 `std::` 前缀。例如，当您看到 `string` 时，可以确定我指的是 `std::string`。
- 不添加虚析构函数：虽然在实际项目中可能有必要添加虚析构函数，但在示例中我通常会省略它们。
- 按值传递参数：在极少数情况下，我会通过值创建和传递参数以避免过多使用 `shared_ptr`、`make_shared` 等智能指针。智能指针增加了另一层复杂性，将其整合到书中介绍的设计模式中留作读者练习。
- 省略某些代码元素：有时我会省略那些为类型功能完整所必需的代码元素（如移动构造函数），因为这些内容会占用太多篇幅。
- 省略 `const` 修饰符：在很多情况下，我会省略本应使用的 `const` 修饰符。常量正确性（const-correctness）通常会导致API表面的分裂和加倍，这在书籍格式中并不理想。

您应该知道，本书中的大部分示例都利用了现代 C++（C++11、C++14、C++17 及以后版本），并通常使用开发者可用的最新 C++ 语言特性。例如，当 C++14 允许我们自动推导返回类型时，您不会看到许多函数签名以 `-> decltype(...)` 结尾。示例并不针对特定的编译器，但如果某些内容在您选择的编译器上无法工作，您需要找到变通方法。

在某些时候，我会引用其他编程语言，如 C# 或 Kotlin。有时，注意其他语言的设计者是如何实现某个特定功能的是有趣的。C++ 并不陌生于从其他语言借用普遍可用的想法：例如，`auto` 和变量声明及返回类型的类型推导的引入在许多其他语言中也存在。

### Developer Tools

本书中的代码示例是为现代 C++ 编译器编写的，无论是 Clang、GCC 还是 MSVC。我通常假设您使用的是最新版本的编译器，因此会利用最新的语言特性。在某些情况下，高级语言示例可能需要为较早版本的编译器降级；而在其他情况下，可能根本无法实现。

至于开发工具，本书并未具体涉及。只要您拥有更新的编译器，应该可以顺利跟随示例进行：大多数示例都是自包含的 .cpp 文件。尽管如此，我想借此机会提醒您，高质量的开发工具如 CLion 或 ReSharper C++ 可以大大提升开发体验。通过投入少量的资金，您可以获得大量额外的功能，这些功能直接转化为编码速度的提高和代码质量的改进。

### Piracy

数字盗版是一个无法回避的事实。新一代人正在成长，他们从未购买过电影或书籍——甚至包括这本书。对此能做的不多。我唯一能说的是，如果您盗版了这本书，您可能不会阅读到最新版本。

在线数字出版的好处之一是我可以随着新版本的C++发布和我进行更多研究而更新这本书。因此，如果您购买了这本书，未来将免费获得随着 C++ 语言和标准库的新版本发布的更新。如果不是这样的话…… 唉，也无可奈何。

## Important Concepts

在我们开始之前，我想简要提及一些本书中将会引用的 C++ 世界的关键概念。

### Important Concepts

嘿，这确实是一种模式！我不知道它是否够格被列为一个独立的设计模式，但在 C++ 世界中，它确实是一种特定的模式。本质上，这个想法很简单：派生类将自身作为模板参数传递给其基类。

```c++
struct Foo : SomeBase<Foo>
{
  ...
}
```

现在，您可能会想知道为什么有人会这样做？原因之一是为了能够在基类的实现中访问一个类型化的 `this` 指针。

例如，假设每一个SomeBase的派生类都实现了用于迭代所需的 `begin()`/`end()` 方法对。您如何在 `SomeBase` 的成员内部迭代该对象呢？直觉上，您可能认为这不可能，因为 `SomeBase` 本身并没有提供 `begin()`/`end()` 接口。但是，如果您使用 CRTP（Curiously Recurring Template Pattern），实际上可以将 `this` 指针转换为派生类类型：

```cpp
template <typename Derived>
struct SomeBase {
  void foo()
  {
    for (auto& item : *static_cast<Derived*>(this))
    {
      ...
    }
  }
}
```

为了具体了解这种做法，请参见第 9 章。

### Mixin Inheritance

在 C++ 中，一个类可以被定义为继承自其自身的模板参数，例如：

```cpp
template <typename T>
struct Mixin : T
{
  ...
}
```

这种方法被称为混入继承（mixin inheritance），它允许类型的层次组合。例如，您可以允许 `Foo<Bar<Baz>> x;` 声明一个变量，该变量的类型实现了所有三个类的特征，而无需实际构造一个新的`FooBarBaz` 类型。

为了具体了解这种做法，请参见第9章。

### Properties

属性无非是一个（通常是私有的）字段以及 getter 和 setter 的组合。在标准 C++ 中，一个属性看起来如下：

```cpp
class Person
{
  int age;
public:
  int get_age() const { return age; }
  void set_age(int value) { age = value; }
};
```

许多语言（例如，C#、Kotlin）通过直接将其内置到编程语言中来内部化属性的概念。虽然 C++ 尚未这样做（并且在可预见的未来也不太可能这样做），但有一个非标准的声明限定符叫做 `property`，您可以在大多数编译器（如 MSVC、Clang、Intel）中使用它：

```cpp
class Person
{
  int age_;
public:
  int get_age() const { return age; }
  void set_age(int value) { age = value; }
  __declspec(property(get=get_age, put=set_age)) int age;
};
```

这可以如下使用：

```cpp
Person person;
p.age = 20; // calls p.set_age(20)
```

## The SOLID Design Principles

SOLID 是一个首字母缩略词，代表以下设计原则（及其缩写）：

- 单一职责原则（Single Responsibility Principle, SRP）
- 开放封闭原则（Open-Closed Principle, OCP）
- 里氏替换原则（Liskov Substitution Principle, LSP）
- 接口隔离原则（Interface Segregation Principle, ISP）
- 依赖倒置原则（Dependency Inversion Principle, DIP）

这些原则是由 Robert C. Martin 在 2000 年代初期引入的——实际上，它们只是罗伯特在其书籍和博客中表达的数十条原则中的五条。这五个特定的主题贯穿了模式和软件设计的一般讨论，所以在我们深入探讨设计模式之前（我知道你们都很迫不及待），我们将简要回顾一下 SOLID 原则的内容。

### Single Responsibility Principle

假设您决定记录下您最私密的想法。这个日记有一个标题和若干条目。您可以将其建模如下：

```c++
struct Journal
{
  string title;
  vector<string> entries;

  explicit Journal(const string& title) : title{title} {}
};
```

现在，您可以添加功能，将条目按其在日记中的顺序编号前缀添加到日记中。这很简单：

```c++
void Journal::add(const string& entry)
{
  static int count = 1;
  entries.push_back(boost::lexical_cast<string>(count++) + ": " + entry);
}
```

现在，日记可以如下使用：

```c++
Journal j{"Dear Diary"};
j.add("I cried today");
 j.add("I ate a bug");
```

将此功能作为 `Journal` 类的一部分是有道理的，因为添加日记条目是日记本身需要执行的操作。管理条目是日记的责任，因此任何与此相关的功能都属于其职责范围。

现在假设您决定通过将日记保存到文件中来使其持久化。您将以下代码添加到 `Journal` 类中：

```c++
void Journal::save(const string& filename)
{
  ofstream ofs(filename);
  for (auto& s : entries)
    ofs << s << endl;
}
```

这种做法存在问题。日记的责任是管理日记条目，而不是将它们写入磁盘。如果您将磁盘写入功能添加到 `Journal` 类以及类似的类中，那么一旦持久化方法发生变化（例如，您决定将数据写入云端而不是磁盘），这将需要对每个受影响的类进行大量细微的修改。

我想在这里暂停并强调一点：一种导致您需要在许多类中进行大量细微修改的架构，无论这些类是否相关（例如在一个继承层次结构中），通常是一个代码异味——表明某些地方不太对劲。当然，这确实取决于具体情况：如果您正在重命名一个在上百个地方被使用的符号，我认为这通常是没问题的，因为 ReSharper、CLion 或其他您使用的 IDE 实际上可以让您执行重构，并将更改传播到所有相关位置。但是，当您需要完全重做接口时……那可能会是一个非常痛苦的过程！

因此，我主张持久化是一个独立的关注点，最好是在一个单独的类中表达，例如：

```c++
struct PersistenceManager
{
  static void save(const Journal& j, const string& filename)
  {
    ofstream ofs(filename);
    for (auto& s : j.entries)
      ofs << s << endl;
  }
};
```

这正是单一职责原则（Single Responsibility）的含义：每个类只有一个职责，因此它只有一个需要更改的理由。`Journal`类只应在与条目存储相关的需求发生变化时进行修改——例如，您可能希望每个条目前加上时间戳，那么您会修改 `add()` 函数以实现这一点。另一方面，如果您想要更改持久化机制，这样的改动应当在 `PersistenceManager` 类中进行。

违反单一职责原则（SRP）的一个极端反模式例子被称为“上帝对象”（*God Object*）。上帝对象是一个巨大的类，试图处理尽可能多的关注点，最终成为一个难以处理的单片式怪物。

幸运的是，对于我们来说，上帝对象很容易识别，而且借助源代码控制系统（只需统计成员函数的数量），可以迅速确定负责的开发者并对其进行适当的“惩罚”。

### Open-Closed Principle

假设我们在数据库中有一系列（完全是假设的）产品。每个产品都有颜色和尺寸，并被定义为：

```c++
enum class Color { Red, Green, Blue };
enum class Size { Small, Medium, Large };

struct Product
{
  string name;
  Color color;
  Size size;
};
```

现在，我们希望为给定的一组产品提供某些过滤功能。我们创建一个类似于以下的过滤器：

```c++
struct ProductFilter
{
  typedef vector<Product*> Items;
};
```

现在，为了支持按颜色过滤产品，我们定义一个成员函数来实现这一功能：

```c++
ProductFilter::Items ProductFilter::by_color(Items items, Color color)
{
  Items result;
  for (auto& i : items)
    if (i->color == color)
      result.push_back(i);
  return result;
}
```

我们目前按颜色过滤项目的做法固然不错。代码投入生产后，不幸的是，一段时间后老板进来要求我们也要实现按尺寸过滤的功能。于是我们回到 ProductFilter.cpp，添加以下代码并重新编译：

```c++
ProductFilter::Items ProductFilter::by_color(Items items, Size size)
{
  Items result;
  for (auto& i : items)
    if (i->size == size)
      result.push_back(i);
  return result;
}
```

这看起来像是直接的代码重复，不是吗？为什么不直接编写一个接受谓词（某个函数）的一般方法呢？原因之一可能是不同的过滤形式可以有不同的实现方式：例如，某些记录类型可能是索引过的，需要以特定方式进行搜索；某些数据类型可以在 GPU 上进行搜索，而其他类型则不行。

我们的代码投入生产后，但又一次，老板回来告诉我们现在需要同时按颜色和尺寸进行搜索。所以我们能做的就是再添加一个函数吗？

```c++
ProductFilter::Items ProductFilter::by_color_and_size(Items items, Size size, Color color)
{
  Items result;
  for (auto& i : items)
    if (i->size == size && i->color == color)
      result.push_back(i);
    return result;
}
```

从前面的情景中，我们希望遵循开闭原则（*Open-Closed Principle*），该原则指出一个类型应该对扩展开放，但对修改封闭。换句话说，我们希望过滤功能是可扩展的（可能是在不同的编译单元中），而无需修改现有的代码（以及重新编译已经工作正常并可能已交付给客户的代码）。

我们如何实现这一点呢？首先，我们要概念上分离（SRP！）我们的过滤过程为两部分：一个是过滤器（接收所有项目并只返回其中一些项目的流程），另一个是规范（应用于数据元素的谓词定义）。

我们可以给出一个非常简单的规范接口定义：

```c++
template <typename T>
struct Specification
{
  virtual bool is_satisfied(T* item) = 0;
};
```

在前面的例子中，类型T可以是我们选择的任何类型：它可以是Product，但也可以是其他类型。这使得整个方法具有可重用性。

接下来，我们需要一种基于Specification<T>进行过滤的方法：这是通过定义——您可能已经猜到了——一个 `Filter<T>` 来实现的：

```c++
template <typename T>
struct Filter
{
  virtual vector<T*> filter(vector<T*> items, Specification<T>& spec) = 0;
};
```

我们所做的只是为一个名为 `filter` 的函数指定签名，该函数接收所有项目和一个规范，并返回符合该规范的所有项目。这里假设项目是以 `vector<T*>` 的形式存储的，但实际上您可以传递给 `filter()` 一对迭代器或专门为遍历集合设计的某些自定义接口。遗憾的是，C++ 语言未能标准化枚举或集合的概念，这是其他编程语言（例如 .NET 的IEnumerable）中已有的功能。

基于上述内容，改进后的 `filter` 实现非常简单：

```c++
struct BetterFilter : Filter<Product>
{
  vector<Product*> filter(vector<Product*> items, Specification<Product>& spec) override
  {
    vector<Product*> result;
    for (auto& p : items)
      if (spec.is_satisfied(p))
        result.push_back(p)
    return result;
  }
}
```

再次说明，您可以将传递进来的 `Specification<T>` 视为一个强类型的 `std::function` 等价物，它仅受限于一定数量的可能过滤规范。

现在，是简单部分。为了创建一个颜色过滤器，您可以定义一个 `ColorSpecification`：

```c++
struct ColorSpecification : Specification<Product>
{
  Color color;

  explicit ColorSpecification(const Color color) : color{color} {}

  bool is_satisfied(Product* item) override {
    return item->color == color;
  }
};
```

有了这个规范，并给定一个产品列表，我们现在可以如下进行过滤：

```c++
Product apple{ "Apple", Color::Green, Size::Small };
Product tree{ "Tree", Color::Green, Size::Large };
Product house{ "House", Color::Blue, Size::Large };

vector<Product*> all{ &apple, &tree, &house };

BetterFilter bf;
ColorSpecification green(Color::Green);

auto green_things = bf.filter(all, green);
for (auto& x : green_things)
  cout << x->name << " is green" << endl;
```

上述方法会得到“Apple”和“Tree”，因为它们都是绿色的。现在，我们唯一还没有实现的是按尺寸和颜色进行搜索（或者确切地说，还没有解释如何搜索尺寸或颜色，或混合不同标准）。解决办法是创建一个复合规范（composite specification）。例如，对于逻辑与（AND），您可以如下实现：

```c++
template <typename T>
struct AndSpecification : Specification<T>
{
  Specification<T>& first;
  Specification<T>& second;

  AndSpecification(Specification<T>& first, Specification<T>& second) : first{first}, second{second} {}

  bool is_satisfied(T* item) override
  {
    return first.is_satisfied(item) && second.is_satisfied(item);
  }
};
```

现在，您可以自由地基于更简单的 `Specification` 创建复合条件。重用我们之前创建的绿色规范，查找既为绿色又大的物品变得非常简单：

```c++
SizeSpecification large(Size::Large);
ColorSpecification green(Color::Green);
AndSpecification<Product> green_and_large{ large, green };

auto big_green_things = bf.filter(all, green_and_large);
for (auto& x : big_green_things)
  cout << x->name << " is large and green" << endl;

// Tree is large and green
```

这确实有很多代码！但请记住，得益于C++的强大功能，您可以简单地为两个 `Specification<T>` 对象引入一个 `&&` 运算符，从而使根据两个（或更多！）标准进行过滤的过程变得极其简单：

```c++
template <typename T>
struct Specification
{
  virtual bool is_satisfied(T* item) = 0;

  AndSpecification<T> operator &&(Specification&& other)
  {
    return AndSpecification<T>(*this, other);
  }
};
```

如果您现在避免为尺寸/颜色规范创建额外的变量，复合规范可以简化为一行代码：

```c++
auto green_and_big = ColorSpecification(Color::Green) && SizeSpecification(Size::Large);
```

让我们回顾一下开闭原则（OCP）是什么，以及前面的例子是如何遵循这一原则的。基本上，OCP 指出您不应该需要回到已经编写和测试过的代码并修改它。而这正是这里的情况！我们创建了`Specification<T>` 和 `Filter<T>`，从那时起，我们只需要实现这些接口之一（而不修改接口本身）来实现新的过滤机制。这就是“对扩展开放，对修改封闭”的含义。

### Liskov Substitution Principle

里氏替换原则（Liskov Substitution Principle，LSP），以 Barbara Liskov 的名字命名，指出如果一个接口接受类型为 Parent 的对象，那么它也应该能够接受类型为 Child 的对象而不出现问题。让我们来看一个 LSP 被违反的情况。

这里有一个矩形；它有宽度和高度以及一堆用于计算面积的 getter 和 setter：

```c++
class Rectangle
{
protected:
  int width, height;
public:
  Rectangle(const int width, const int height) : width{width}, height{height} {}

  int get_width() const { return width; }
  virtual void set_width(const int width) { this->width = width; }
  int get_height() const { return height; }
  virtual void set_height(const int height) { this->height = height; }

  int area() const { return width * height; }
};
```

现在，假设我们创建了一种特殊的矩形，叫做 Square（正方形）。这个对象重写了 `setter` 方法来同时设置宽度和高度：

```c++
class Square : public Rectangle
{
public:
  Square(int size) : Rectangle(size, size) {}
  void set_width(const int width) override {
    this->width = height = width;
  }
  void set_height(const int height) override {
    this->height = width = height;
  }
};
```

这种方法存在潜在问题。目前您可能还看不出来，因为它看起来确实非常无辜：`setter` 方法只是同时设置两个维度，这怎么可能出错呢？然而，如果我们考虑前面的内容，我们可以很容易地构建一个接受 `Rectangle` 的函数，当传入一个 `Square` 时，这个函数可能会出现问题：

```c++
void process(Rectangle& r)
{
  int w = r.get_width();
  r.set_height(10);

  cout << "expected area = " << (w * 10) << ", got " << r.area() << endl;
}
```

前面的函数将公式 `Area = Width × Height` 视为一个不变量。它获取宽度，设置高度，并合理地期望乘积等于计算出的面积。但是，用 Square 调用该函数会导致不匹配：

```c++
Square s{5};
process(s); // expected area = 50, got 25
```

从这个例子中可以得出的结论是（我承认这个例子有点人为构造），`process()` 函数通过完全无法接受派生类型`Square` 来替代基类型 `Rectangle`，从而违反了里氏替换原则（LSP）。如果给它传入一个 `Rectangle` 对象，一切正常，因此问题可能需要一段时间才会在测试中显现出来（或是在生产环境中——希望不会如此！）。

解决方案有哪些呢？其实有很多。我个人认为，类型 `Square` 根本就不应该存在：相反，我们可以创建一个工厂（参见第3章），该工厂既能创建矩形也能创建正方形：

```c++
struct RectangleFactory
{
  static Rectangle create_rectangle(int w, int h);
  static Rectangle create_square(int size);
};
```

您可能还需要一种方法来检测一个Rectangle实际上是否是正方形：

```c++
bool Rectangle::is_square() const
{
  return width == height;
}
```

在这种情况下，所谓的“核选项”是在 `Square` 的 `set_width()` 和 `set_height()` 方法中抛出一个异常，指出这些操作不被支持，并且应该使用 `set_size()` 代替。然而，这违反了最小惊讶原则（principle of least surprise），因为您会期望调用 `set_width()` 能够产生有意义的改变……我说得对吗？

### Interface Segregation Principle

OK，这里有一个虽然人为构造但仍然适合说明问题的例子。假设您决定定义一个多功能打印机：一种可以打印、扫描并且还能传真文档的设备。于是您像这样定义它：

```c++
struct MyFavouritePrinter /* : IMachine */
{
  void print(vector<Document*> docs) override;
  void fax(vector<Document*> docs) override;
  void scan(vector<Document*> docs) override;
};
```

这没问题。现在，假设您决定定义一个接口，该接口需要由所有计划制造多功能打印机的开发者实现。于是您可以使用您最喜欢的 IDE 中的“提取接口”功能，得到如下结果：

```c++
struct IMachine
{
  void print(vector<Document*> docs) = 0;
  void fax(vector<Document*> docs) = 0;
  void scan(vector<Document*> docs) = 0;
};
```

这是一个问题。之所以这是个问题，是因为某些实现这个接口的开发者可能只需要打印功能，而不需要扫描或传真功能。然而，您却强制他们实现这些额外的功能：虽然这些方法可以都是空操作（no-op），但这样做显然是多余的。

因此，ISP（接口隔离原则）建议将接口拆分，以便实现者可以根据自己的需求选择性地实现接口。由于打印和扫描是不同的操作（例如，扫描仪无法打印），我们为这些操作定义单独的接口：

```c++
struct IPrinter
{
  virtual void print(vector<Document*> docs) = 0;
};

struct IScanner
{
  virtual void scan(vector<Document*> docs) = 0;
};
```

然后，打印机或扫描仪只需实现所需的功能：

```c++
struct Printer : IPrinter
{
  void print(vector<Document*> docs) override;
};

struct Scanner : IScanner
{
  void scan(vector<Document*> docs) override;
};
```

现在，如果我们确实想要一个 `IMachine` 接口，我们可以将其定义为前述接口的组合：

```c++
struct IMachine : IPrinter, IScanner /* IFax and so on */
{
};
```

当您要在具体的多功能设备中实现这个接口时，这就是要使用的接口。例如，您可以使用简单的委托来确保 `Machine` 重用特定的 `IPrinter` 和 `IScanner` 所提供的功能：

```c++
struct Machine : IMachine
{
  IPrinter& printer;
  IScanner& scanner;

  Machine(IPrinter& printer, IScanner& scanner) : printer{printer}, scanner{scanner}
  {
  }

  void print(vector<Document*> docs) override {
    printer.print(docs);
  }

  void scan(vector<Document*> docs) override {
    scanner.scan(docs);
  }
};
```

所以，简单回顾一下，这里的思路是将复杂接口的部分功能分离成独立的接口，以避免强迫实现者去实现他们实际上并不需要的功能。每当您为某个复杂的应用程序编写插件，并被要求实现一个带有20个令人困惑的函数的接口，其中许多函数只是空操作（no-ops）或返回 `nullptr` 时，很可能 API 的设计者违反了接口隔离原则（ISP）。

### Dependency Inversion Principle

依赖倒置原则（DIP）的原始定义包括以下两点：

*A. 高层模块不应该依赖于低层模块。两者都应该依赖于抽象。*

这句话的基本含义是，如果您关注的是日志记录功能，您的报告组件不应该依赖于具体的 `ConsoleLogger` 实现，而是可以依赖于一个 `ILogger` 接口。在这种情况下，我们将报告组件视为高层（更接近业务领域），而日志记录作为基础性关注点（类似于文件 I/O 或线程处理，但不完全相同）被视为低层模块。

*B. 抽象不应该依赖于细节。细节应该依赖于抽象。*

这再次强调了依赖于接口或基类比依赖于具体类型更好。希望这个陈述的真实性是显而易见的，因为这种方法支持更好的可配置性和可测试性——前提是您使用了一个良好的框架来处理这些依赖关系。

现在主要的问题是：如何实际实现上述原则？这确实需要更多的工作，因为现在您需要明确声明，例如，`Reporting` 依赖于 `ILogger`。您可以这样表达这种依赖关系：

```c++
class Reporting
{
  ILogger& logger;
public:
  Reporting(const ILogger& logger) : logger{logger} {}
  void prepare_report()
  {
    logger.log_info("Preparing the report");
    ...
  }
};
```

现在的问题是，为了初始化上述类，您需要显式地调用 `Reporting{ConsoleLogger{}}` 或类似的方法。如果`Reporting` 依赖于五个不同的接口，或者 `ConsoleLogger` 本身也有依赖关系，那该怎么办？虽然可以通过编写大量代码来管理这些依赖关系，但有更好的方法。

现代、流行且时尚的做法是使用依赖注入（*Dependency Injection*, DI）：这实际上意味着使用如 Boost.DI 这样的库来自动满足特定组件的依赖需求。

让我们考虑一个例子，假设有一辆汽车，它有一个引擎，同时也需要写入日志。可以说，汽车依赖于这两者。首先，我们可以这样定义引擎：

```c++
struct Engine
{
  float volume = 5;
  int horse_power = 400;

  friend ostream& operator<< (ostream& os, const Engine& obj)
  {
    return os
      << "volume: " << obj.volume
      << " horse_power: " << obj.horse_power;
  } // thanks, ReSharper
};
```

现在，由我们决定是否要提取一个 `IEngine` 接口并将其提供给汽车。也许我们会这样做，也许不会，这通常是一个设计决策。如果您预见到会有一个引擎的层级结构，或者您预计在测试时需要一个 `NullEngine`（参见第19章），那么是的，您确实需要抽象出这些接口。

无论如何，我们也希望有日志记录功能，并且由于日志记录可以通过多种方式进行（控制台、电子邮件、短信、信鸽邮递等），我们很可能想要有一个 `ILogger` 接口：

```c++
struct ILogger
{
  virtual ~ILogger() {}
  virtual void Log(const string& s) = 0;
};
```

以及某种具体的实现：

```c++
struct ConsoleLogger : ILogger
{
  ConsoleLogger() {}

  void Log(const string& s) override
  {
    cout << "LOG: " << s.c_str() << endl;
  }
};
```

现在，我们即将定义的汽车依赖于引擎和日志记录组件两者。我们确实需要这两个组件，但如何存储它们则取决于我们的选择：我们可以使用指针、引用、`unique_ptr`、`shared_ptr` 或其他方式。我们将把这两个依赖组件定义为构造函数的参数：

```c++
struct Car
{
  unique_ptr<Engine> engine;
  shared_ptr<ILogger> logger;

  Car(unique_ptr<Engine> engine, const shared_ptr<ILogger>& logger) : engine{move(engine)}, logger{logger}
  {
    logger->Log("making a car");
  }

  friend ostream& operator<<(ostream& os, const Car& obj)
  {
    return os << "car with engine: " << *obj.engine;
  }
};
```

现在，您可能期望在初始化Car时看到 `make_unique` 或 `make_shared` 的调用。但我们不会这样做。相反，我们将使用 Boost.DI 来进行依赖注入。首先，我们将定义一个绑定，将`ILogger` 绑定到 `ConsoleLogger`；这基本上意味着“任何时候有人请求 `ILogger`，就给他们一个 `ConsoleLogger`”。

```c++
auto injector = di::make_injector(
    di::bind<ILogger>().to<ConsoleLogger>()
);
```

既然我们已经配置好了注入器，现在可以使用它来创建一辆汽车：

```c++
auto car = injector.create<shared_ptr<Car>>();
```

上述代码创建了一个指向完全初始化的 `Car` 对象的 `shared_ptr<Car>`，这正是我们所期望的结果。这种方法的优点在于，要更改使用的日志记录器类型，我们可以在一个地方（`bind` 调用）进行修改，所有出现 `ILogger` 的地方现在都可以使用我们提供的其他日志记录组件。这种方法也帮助我们进行单元测试，并允许我们使用存根（stubs）或空对象模式（Null Object pattern）来代替模拟对象（mocks）。

### Time for Patterns!

在理解了SOLID设计原则之后，我们已经准备好来审视设计模式本身了。系好安全带；这将是一段漫长（但希望不会无聊）的旅程！
