# Prototype

思考一下您每天使用的物品，比如汽车或手机。很可能这些产品并不是从零开始设计的；相反，制造商选择了现有的设计，进行了一些改进，使其在视觉上区别于旧设计（这样人们可以炫耀），然后开始销售新产品，淘汰旧产品。这是一种自然的状态，在软件世界中，我们也遇到类似的情况：有时，我们并不想从头创建一个完整的对象（工厂模式和构建器模式可以帮助实现这一点），而是想要取用一个预先构建好的对象，要么直接使用它的副本（这很简单），或者对其进行一些定制。

这就引出了原型（Prototype）的概念：一个我们可以复制、自定义这些副本并加以使用的模型对象。原型模式的挑战实际上在于复制部分；其他的一切都相对简单。

## Object Constrution

大多数对象的构建是通过构造函数完成的。但是，如果您已经有一个配置好的对象，为什么不直接复制这个对象来代替创建一个一模一样的新对象呢？这在您不得不应用构建器模式来简化分步对象构建时尤其相关。

让我们考虑一个简单的例子，但这个例子能清楚地展示复制的过程：

```c++
Contact john{ "John Doe", Address{"123 East Dr", "London", 10 } };
 Contact jane{ "Jane Doe", Address{"123 East Dr", "London", 11 } };
```

您可以看到这里发生的情况。John 和 Jane 在同一栋楼工作，但在不同的办公室。许多其他人也可能在伦敦的 123 East Dr 工作，那么如果我们想避免地址的重复初始化，我们应该怎么做呢？

事实上，原型模式（Prototype pattern）的核心就是对象复制。当然，我们并没有一种统一的方式来实际复制一个对象，但有几种选择，我们可以选择其中一些方法。

## Ordinary Duplication

如果您复制的是一个值，并且您正在复制的对象通过值存储所有内容，则不会有太大问题。例如，如果您将前面例子中的 Contact 和 Address 定义为如下：

```c++
struct Address
{
  string street, city;
  int suite;
};
struct Contact
{
  string name;
  Address address;
};
```

在编写类似的内容时，完全没有问题：

```c++
// here is the prototype:
Contact worker{"", Address{"123 East Dr", "London", 0}};

// make a copy of prototype and customize it
Contact john = worker;
john.name = "John Doe";
john.address.suite = 10;
```

在实践中，这种情况相当少见。在许多情况下，内部的 `Address` 对象可能会是一个指针：

```c++
struct Contact
{
  string name;
  Address *address=; // pointer (or e.g., shared_ptr)
};
```

这给事情带来了麻烦，因为现在 `Contact john = prototype;` 这一行代码复制了指针，导致 john 和 prototype 以及 prototype 的每一个其他副本都共享同一个地址。而这绝对是我们不希望看到的情况。

##  Duplication via Copy Construction

避免复制问题的最简单方法是确保构成对象的所有部分（在本例中是 `Contact` 和 `Address`）都定义了拷贝构造函数。例如，如果我们采用通过拥有指针来存储地址的想法，即：

```c++
struct Contact
{
  string name;
  Address* address;
};
```

那么我们需要创建一个复制构造函数。实际上，在我们的情况下，有两种方法可以实现这一点。直接的方法看起来会像这样：

```c++
Contact(const Contact& other)
  : name{other.name}
  // , address{ new Address{*other.address} }
{
  address = new Address(
    other.address->street,
    other.address->city,
    other.address->suite
  );
}
```

不幸的是，前面的方法不够通用。它在这个特定情况下当然可以工作（前提是 `Address` 有一个初始化其所有成员的构造函数），但如果 `Address` 决定将其街道部分拆分为包含街道名称、门牌号和附加信息的对象呢？那么我们又会遇到相同的复制问题。

在这里一个合理的选择是也在 `Address` 上定义一个拷贝构造函数。在我们的情况下，这相当简单：

```c++
Address(const string& street, const string& city, const int suite)
  : street{street}, city{city}, suite{suite} {}
```

现在我们可以重写 `Contact` 的构造函数以重用这个拷贝构造函数：

```c++
Contact(const Contact& other)
  : name{other.name}
  , address{ new Address{*other.address} }
  {}
```

请注意，请注意，如果您使用 ReSharper 的复制和移动操作生成器，它还会为您提供 `operator=`，在我们的情况下，它将被定义为：

```c++
Contact& operator=(const Contact& other)
{
  if (this == &other)
    return *this;
  name = other.name;
  address = other.address;
  return *this;
}
```

这样好多了。现在，我们可以像以前一样构建一个原型，然后重用它：

```c++
Contact worker{"", new Address{"123 East Dr", "London", 0}};
Contact john{worker}; // or: Contact john = worker;
john.name = "John";
john.suite = 10;
```

这种方法确实有效，而且效果很好。这里唯一真正的问题，也是不容易解决的问题，是实现所有这些复制构造函数所需的额外工作量。诚然，像 ReSharper 这样的工具可以快速处理大多数场景，但仍然有许多需要注意的地方。例如，您认为如果我这样写会发生什么：

```c++
Contact john = worker;
```

如果忘记了为 `Address` 实现拷贝赋值运算符（但为 `Contact` 实现了），那么程序仍然会编译通过。您说得对，对于拷贝构造函数来说情况稍好一些，因为如果您尝试调用一个不存在的拷贝构造函数，会得到一个错误；而 `operator=` 即使没有正确指定操作也会普遍存在。

这里还有另一个问题：假设您开始使用类似双重指针（例如 `void**`）或 `unique_ptr` ？即使有所有这些工具的“魔法”，像 ReSharper 和 CLion 这样的工具在此时也不太可能生成正确的代码，因此对这些类型快速生成代码可能并不是总是最好的主意。

您可以通过坚持使用复制构造函数而不生成复制赋值运算符来减少一些复杂性。另一种选择是放弃复制构造函数，转而采用其他方式，例如：

```c++
template <typename T> struct Cloneable
{
  virtual T clone() const = 0;
}
```

然后继续实现这个接口，并在需要实际副本时调用 `prototype.clone()`。这实际上比拷贝构造函数或拷贝赋值运算符更好地传达了意图。

无论您选择哪种选项，这里的要点是这种方法确实有效，但如果您的对象图非常复杂，它可能会变得有些繁琐。

## Serialization

其他编程语言的设计者也遇到了同样的问题，即必须显式地定义整个对象图的复制操作，并迅速意识到一个类需要“简单可序列化”——也就是说，默认情况下，您应该能够将一个类写入文件，例如，而无需为类添加任何特性（好吧，最多可能只需要一两个属性）。

这与当前的问题有何关联呢？因为如果您可以将某个对象序列化到文件或内存中，那么之后就可以反序列化它，同时保留所有信息，包括所有依赖对象。这样做不是更方便吗？嗯……

遗憾的是，与其他编程语言不同，C++ 在序列化方面并没有提供免费的午餐。我们无法直接将一个复杂对象图序列化到文件中。为什么不能呢？因为在其他编程语言中，编译后的二进制文件不仅包含可执行代码，还包含大量元数据，而序列化是通过一种称为反射的功能实现的——这一功能目前在 C++ 中尚不可用。

如果我们想要实现序列化，就像显式的复制操作一样，我们也需要自己来实现。幸运的是，我们不必手动处理位或者思考如何序列化 `std::string`，而是可以使用一个现成的库叫做 Boost.Serialization 来为我们处理部分工作。以下是如何为 `Address` 类型添加序列化支持的一个示例：

```c++
struct Address
{
  string street;
  string city;
  int suite;
private:
  friend class boost::serialization::access;
  template<class Ar>
  void serialization(Ar& ar, const unsigned int version)
  {
    ar& street;
    ar& city;
    ar& suite;
  }
};
```

这看起来可能有点反直觉，但最终结果是我们已经使用 `&` 运算符指定了 `Address` 中所有需要写入保存对象位置的部分。请注意，上述代码是用于保存和加载数据的成员函数。可以告诉 Boost 在保存和加载时执行不同的操作，但这与我们的原型需求关系不大。

现在，我们也需要对 `Contact` 类型进行相同的处理。接下来是为 `Contact` 添加序列化支持的代码：

```c++
struct Contact
{
  string name;
  Address* address = nullptr;
private:
  friend class boost::serialization::access;
  template<class Ar>
  void serialization(Ar& ar, const unsigned int version)
  {
    ar& name;
    ar& address; // no *
  }
};
```

上述 `serialize()` 函数的结构或多或少是相同的，但请注意一个有趣的地方：我们并没有通过 `ar & *address` 来访问地址，而是仍然将其序列化为 `ar & address`，不解除引用指针。Boost 足够智能，能够理解发生了什么，并且即使 `address` 被设置为 `nullptr`，也能正确地进行序列化和反序列化。

因此，如果您想以这种方式实现原型模式，您需要在对象图中可能出现的每一个类型上实现 `serialize()` 方法。但如果您这样做了，现在您可以定义一种通过序列化/反序列化来克隆对象的方法：

```c++
auto clone = [](const Contact& c)
{
  // 1. Serialize the contact
  ostringstream oss;
  boost::archive::text_oarchive oa(oss);
  oa << c;
  string s = oss.str();

  // 2. Deserialize the contact
  istringstream iss(oss.str());
  bosst::archive::text_iarchive ia(iss);
  Contact result;
  ia >> result;
  return result;
};
```

现在，有了一个名为john的联系人，您可以简单地编写如下代码：

```c++
Contact jane = clone(john);
jane.name = "Jane"; // and so on
```

然后您可以随心所欲地自定义jane。

## Prototype Factory

