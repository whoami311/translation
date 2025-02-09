# Factories

*I had a problem and tried to use Java, now I have a ProblemFactory.*

-Old Java joke.

本章同时涵盖了两个GoF（设计模式的经典著作《设计模式：可复用面向对象软件的基础》中的）模式：工厂方法（Factory Method）和抽象工厂（Abstract Factory）。这两个模式密切相关，因此我们将一起讨论它们。

## 场景

让我们从一个激励性的例子开始。假设您想要存储关于笛卡尔空间中一个点（Point）的信息。于是，您可能会实现如下内容：

```c++
struct Point
{
  Point(const float x, const float y)
    : x{x}, y{y} {}
  float x, y; // strictly Cartesian
}
```

到目前为止，一切顺利。但现在，您还想使用极坐标来初始化这个点。因此，您需要另一个具有以下签名的构造函数：

```c++
Point(const float r, const float theta)
{
  x = r * cos(theta);
  y = r * sin(theta);
}
```

但不幸的是，您已经有一个带有两个浮点数参数的构造函数，因此不能有另一个相同签名的构造函数。在这种情况下，您无法仅通过构造函数重载来区分笛卡尔坐标和极坐标初始化。那么，您可以怎么做呢？一种方法是引入一个枚举：

```c++
enum class PointType
{
  cartesian,
  polar
};
```

然后可以向点的构造函数添加另一个参数：

```c++
Point(float a, float b, PointType type = PointType::cartesian)
{
  if (type == PointType::cartesian)
  {
    x = a;
    y = b;
  }
  else
  {
    x = a * cos(b);
    y = a * sin(b);
  }
}
```

请注意，前两个参数的名称被改为 a 和 b：我们不能再告诉用户这些值应该来自哪个坐标系统。与使用 x、y、rho 和 theta 来传达意图相比，这是一种表达力上的明显损失。

总体而言，我们的构造函数设计是可用的，但不太优雅。让我们看看是否可以改进它。

##  Factory Method

构造函数的问题在于它的名称总是与类型名称匹配。这意味着我们无法在其中传达任何额外信息，不像普通函数那样可以做到。此外，由于名称总是相同的，我们不能有两个重载版本，一个接受 x, y，另一个接受 r, theta。

那么我们可以做什么呢？我们可以将构造函数设为 `protected`，然后暴露一些静态函数来创建新的点。

```c++
struct Point
{
protected:
  Point(const float x, const float y)
    : x{x}, y{y} {}
public:
  static Point NewCartesian(float x, float y)
  {
    return { x, y };
  }
  static Point NewPolar(float r, float theta)
  {
    return { r * cos(theta), r * sin(theta) };
  }
  // other members here
};
```

上述每个静态函数被称为工厂方法（Factory Method）。它所做的只是创建一个 `Point` 对象并返回它，其优点在于方法名称及参数名称都能清晰地传达所需的是哪种坐标。

现在，要创建一个点，您只需编写：

```c++
auto p = Point::NewPolar(5, M_PI_4);
```

从上述内容中，我们可以清楚地推断出我们正在使用极坐标创建一个新的点，其中 `r = 5` 和 `theta = π/4`。

## Factory

就像构建器模式一样，我们可以将所有创建 `Point` 的函数从 `Point` 类中移出到一个单独的类中，即所谓的工厂（Factory）类。首先，我们重新定义 `Point` 类：

```c++
struct Point
{
  float x, y;
  friend class PointFactory;
private:
  Point(float x, float y) : x(x), y(y){}
};
```

这里有两点值得注意：

- `Point` 的构造函数是 `private`，因为我们不希望任何人直接调用它。这并不是一个严格的要求，但将其设为公有会带来一些歧义，因为它会给用户提供两种不同的对象构造方式。
- `Point` 将 `PointFactory` 声明为友元类。这是故意为之，以便 `Point` 的私有构造函数对工厂类可用——没有这一点，工厂将无法实例化对象！这意味着这两种类型是同时创建的，而不是工厂类在之后被创建。

现在，我们只需在一个名为 `PointFactory` 的单独类中定义我们的 `NewXxx()` 函数：

```c++
struct PointFactory
{
  static Point NewCartesian(float x, float y)
  {
    return Point{ x, y };
  }
  static Point NewPolar(float r, float theta)
  {
    return Point{ r * cos(theta), r * sin(theta) };
  }
};
```

就是这样——我们现在有了一个专门设计用于创建 `Point` 实例的类，可以如下使用：

```c++
auto my_point = PointFactory::NewCartesian(3, 4);
```

## Inner Factory

内部工厂（inner factory）只是创建类型的内部类中的一个工厂。公平地说，内部工厂通常是 C#、Java 等不支持 `friend` 关键字的语言的产物，但在 C++ 中也没有理由不能使用内部工厂。

内部工厂存在的原因是因为内部类可以访问外部类的 `private` 成员，反之亦然，外部类也可以访问内部类的私有成员。这意味着我们的 `Point` 类也可以如下定义：

```c++
struct Point
{
private:
  Point(float x, float y) : x(x), y(y) {}

  struct PointFactory
  {
  private:
    PointFactory() {}
  public:
    static Point NewCartesian(float x, float y)
    {
      return { x, y };
    }
    static Point NewPolar(float r, float theta)
    {
      return { r * cos(theta), r * sin(theta) };
    }
  };
public:
  float x, y;
  static PointFactory Factory;
};
```

好的，那么这里发生了什么呢？我们将工厂直接嵌入到了它所创建的类中。如果一个工厂只与单一类型打交道，这样做是方便的；但如果一个工厂依赖于多个类型（尤其是当它还需要访问这些类型的 `private` 成员时），这样做就不那么方便了。

您会注意到我在这里耍了个小聪明：整个工厂被放在了一个 `private` 块内，并且其构造函数也被标记为 `private`。实际上，即使我们可以将这个工厂公开为 `Point::PointFactory`，这听起来也相当复杂。相反，我定义了一个名为 `Factory` 的静态成员。这使得我们可以更简便地使用工厂：

```c++
auto pp = Point::Factory.NewCartesian(2, 3);
```

如果出于某种原因，您不喜欢混合使用 `::` 和 `.`，当然可以修改代码以确保处处使用 `::`。有两种方法可以做到这一点：

- 将工厂类设为公有，这样就可以编写
`Point::PointFactory::NewXxx(...)`
- 如果您不喜欢上述方法中 `Point` 出现两次，可以使用 `typedef PointFactory Factory`，然后简单地编写 `Point::Factory::NewXxx(...)`。这可能是能够想到的最合理的语法。或者直接将内部工厂命名为 `Factory`，这几乎一劳永逸地解决了问题……除非您后来决定将其提取出来。

是否拥有内部工厂的决定在很大程度上取决于您喜欢如何组织代码。然而，从原始对象暴露工厂极大地提高了 API 的可用性。如果我找到一个名为 `Point` 且构造函数为 `private` 的类型，我如何知道这个类是打算如何使用的呢？除非 `Person::` 在代码补全列表中给我提供了有意义的内容，否则我是无法知道的。

## Abstract Factory

到目前为止，我们一直在讨论单个对象的构建。有时，您可能会参与到一系列相关对象家族的创建中。这种情况实际上相当少见，因此与工厂方法和简单工厂模式不同，抽象工厂模式（Abstract Factory）通常只在复杂的系统中出现。尽管如此，我们仍然需要讨论它，主要是出于历史原因。

这里有一个简单的场景：假设您在一家咖啡馆工作，该咖啡馆提供茶和咖啡。这两种热饮是通过完全不同的设备制作的，我们可以将这些设备建模为某种形式的工厂。茶和咖啡都可以提供热的或冷的版本，但让我们专注于热饮。首先，我们可以定义什么是 `HotDrink`：

```c++
struct HotDrink
{
  virtual void prepare(int volume) = 0;
};
```

函数 `prepare` 是我们用来准备特定容量的热饮的方法。例如，对于类型 `Tea`，它将被实现为：

```c++
struct Tea : HotDrink
{
  void prepare(int volume) override
  {
    cout << "Take tea bag, boil water, pour " << volume <<
      "ml, add some lemon" << endl;
  }
};
```

同样地，对于 `Coffee` 类型也是如此。在这一点上，我们可以编写一个假设的 `make_drink()` 函数，该函数将接受饮品的名称并制作相应的饮品。鉴于只有一组离散的情况，这样的实现可能会显得相当繁琐：

```c++
unique_ptr<HotdDrink> make_drink(string type)
{
  unique_ptr<HotDrink> drink;
  if (type == "tea")
  {
    drink = make_unique<Tea>();
    drink->prepare(200);
  }
  else
  {
    drink = make_unique<Coffee>();
    drink->prepare(50);
  }
  return drink;
}
```

现在，记住，不同的饮品是由不同的设备制作的。在我们的情况下，我们关注的是热饮，我们将通过恰如其分命名的 `HotDrinkFactory` 来建模：

```c++
struct HotDrinkFactory
{
  virtual unique_ptr<HotDrink> make() const = 0;
};
```

这个类型实际上是一个抽象工厂（Abstract Factory）：它是一个具有特定接口的工厂，但它本身是抽象的，这意味着虽然它可以作为函数参数出现，但我们需要具体的实现来实际制作饮品。例如，在制作咖啡的情况下，我们可以编写：

```c++
struct CoffeeFactory : HotDrinkFactory
{
  unique_ptr<HotDrink> make() const override
  {
    return make_unique<Coffee>();
  }
};
```

同样的逻辑也适用于 `TeaFactory`，如前所述。现在，假设我们想要定义一个更高级别的接口来制作不同的饮品，无论是热饮还是冷饮。我们可以创建一个名为 `DrinkFactory` 的类型，该类型内部包含对各种可用工厂的引用：

```c++
class DrinkFactory
{
  map<string, unique_ptr<HotDrinkFactory>> hot_factories;
public:
  DrinkFactory()
  {
    hot_factories["coffee"] = make_unique<CoffeeFactory>();
    hot_factories["tea"] = make_unique<TeaFactory>();
  }

  unique_ptr<HotDrink> make_drink(const string& name)
  {
    auto drink = hot_factories[name]->make();
    drink->prepare(200); // oops!
    return drink;
  }
};
```

在这里，我假设我们希望根据饮品的名称而不是某个整数或枚举成员来分发饮品。我们只需创建一个字符串与相应工厂之间的映射：实际的工厂类型是 `HotDrinkFactory`（我们的抽象工厂），并且我们通过智能指针存储它们，而不是直接存储（这有道理，因为我们希望防止对象切割）。

现在，当有人想要一杯饮品时，我们找到相关的工厂（想象一下咖啡店服务员走到正确的机器前），创建饮品，准备所需的确切容量（在前面的例子中我将其设置为常量；您可以自由地将其提升为参数），然后返回相应的饮品。这就是全部的过程。

## Functional Factory

我还想提到最后一件事：当我们通常使用 “factory” 这个词时，我们通常指的是以下两种情况之一：

- 一个知道如何创建对象的类
- 一个在被调用时创建对象的函数

第二种选项不仅仅是在经典意义上所说的工厂方法（Factory Method）。如果有人将返回类型为 `T` 的 `std::function` 传递给某个函数，这通常被称为工厂（Factory），而不是工厂方法。这可能看起来有点奇怪，但考虑到 “Method” 一词与 “Member Function” 同义，这就更有意义了。

幸运的是，函数可以存储在变量中，这意味着我们可以不仅限于存储指向工厂的指针（如我们在之前的DrinkFactory中所做的那样），而是可以内部化准备精确200毫升液体的过程。这是通过从工厂切换到简单地使用函数块来实现的，例如：

```c++
class DrinkWithVolumeFactory
{
  map<string, function<unique_ptr<HotDrink>()>> factories;
public:
  DrinkWithVolumeFactory()
  {
    factories["tea"] = [] {
      auto tea = make_unique<Tea>();
      tea->prepare(200);
      return tea;
    }; // similar for Coffee
  }
};
```

当然，采用了这种方法后，我们现在简化为直接调用存储的工厂函数，也就是说：

```c++
inline unique_ptr<HotDrink>
DrinkWithVolumeFactory::make_drink(const string& name)
{
  return factories[name]();
}
```

然后这可以像之前一样被使用。

## 总结

让我们回顾一下术语：

- 工厂方法（Factory Method） 是一个充当创建对象方式的类成员。它通常替代构造函数。
- 工厂（Factory） 通常是知道如何构建对象的独立类，尽管如果您传递了一个构建对象的函数（如 `std::function` 或类似物），这个参数也被称为工厂。
- 抽象工厂（Abstract Factory） 如其名称所示，是一个可以被具体类继承以提供一系列类型家族的抽象类。抽象工厂在实际应用中较为罕见。

工厂相较于构造函数调用有几个关键优势：

- 工厂可以说“不”，即它可以返回 `nullptr` 而不是实际的对象。
- 工厂的方法命名更灵活且不受限制，不像构造函数那样受限于类名。
- 单个工厂可以创建多种不同类型的对象。
- 工厂可以表现出多态行为，实例化一个类并通过基类的引用或指针返回它。
- 工厂可以实现缓存和其他存储优化；它也是诸如对象池或单例模式（Singleton pattern）等方法的自然选择（更多内容将在第五章讨论）。

工厂与构建器（Builder）的不同之处在于，使用工厂时，您通常是一次性创建一个对象，而使用构建器时，您是分步骤地提供信息来逐步构建对象。