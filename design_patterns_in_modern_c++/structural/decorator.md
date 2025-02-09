# Decorator

假设您正在处理一位同事编写的类，并且您想扩展该类的功能。您如何在不修改原始代码的情况下实现这一点？一种方法是继承：您创建一个派生类，添加所需的功能，甚至可能 `override`某些方法，这样就可以满足需求了。

然而，这种方法并不总是可行，原因有很多。例如，通常您不会从 `std::vector` 继承，因为它没有虚析构函数，或者从 `int` 继承（这根本不可能）。但继承行不通的最关键原因是当您需要多个增强功能时，并且您希望将这些增强功能分开，因为根据单一职责原则（Single Responsibility Principle），每个类应该只负责一项功能。

装饰器模式（Decorator pattern）允许我们在不修改原始类型（符合开闭原则 Open-Closed Principle）或导致派生类型数量激增的情况下增强现有类型。

## 场景

让我解释一下我所说的多个增强功能是什么意思：假设您有一个名为 `Shape` 的类，并创建了两个继承者类，分别叫做 `ColoredShape` 和 `TransparentShape`——同时您还需要考虑到有人会想要一个既有颜色又有透明度的形状，即 `ColoredTransparentShape`。因此，为了支持这两个增强功能，我们生成了三个类；如果我们有三个增强功能，则需要七个（7！）不同的类。别忘了我们实际上还想要不同的形状（如 Square、Circle 等）——这些类要从哪个基类继承呢？如果有三个增强功能和两种不同的形状，类的数量将增加到 14 个。显然，这是一种难以管理的情况——即使您正在使用代码生成工具！

让我们实际为这个场景编写一些代码。假设我们定义了一个抽象类叫做Shape：

```c++
struct Shape
{
  virtual string str() const = 0;
};
```

在前述的类中，`str()` 是一个虚拟函数，我们将使用它来提供特定形状的文本表示。

我们现在可以使用这个接口实现诸如 `Circle` 或 `Square` 等形状：

```c++
struct Circle : public Shape
{
  float radius;

  explicit Circle(const float radius)
    : radius{radius} {}

  void resize(float factor) { radius *= factor; }

  string str() const override
  {
    ostringstream oss;
    oss << "A circle of radius " << radius;
    return oss.str();
  }
}; // Square implementation omitted
```

我们已经知道，仅靠普通的继承并不能为我们提供一种有效的方式来增强形状的功能，因此我们必须转向组合——这就是装饰器模式用来增强对象的机制。实际上，有两种不同的方法来实现这一点，以及其他一些模式需要讨论：

- 动态组合允许您在运行时通过传递引用的方式进行组合。它提供了最大的灵活性，因为组合可以根据用户的输入等在运行时发生。
- 静态组合意味着对象及其增强功能是在编译时通过使用模板组成的。这意味着对象的确切增强集需要在编译时刻就知道，因为之后无法修改。

如果上述内容听起来有些晦涩难懂，别担心——我们将以动态和静态两种方式实现装饰器模式，所有内容很快就会变得清晰起来。

##  Dynamic Decorator

假设我们想要通过添加一些颜色来增强形状。我们将使用组合而不是继承来实现一个 `ColoredShape`，它只是接收一个已经构造好的 `Shape` 的引用并对其进行增强：

```c++
struct ColoredShape : Shape
{
  Shape& shape;
  string color;

  ColoredShape(Shape& shape, const string& color)
    : shape{shape}, color{color} {}

  string str() const override;
  {
    ostringstream oss;
    oss << shape.str() << " has the color " << color;
    return oss.str();
  }
};
```

如您所见，`ColoredShape` 本身是一个 `Shape`。通常您会像这样使用它：

```c++
Circle circle{0.5f};
ColoredShape redCircle{circle, "red"};
cout << redCircle.str();
// prints "A circle of radius 0.5 has the color red"
```

如果我们现在想要另一个增强功能，为形状添加透明度，这也是非常简单的：

```c++
struct TransparentShape : Shape
{
  Shape& shape;
  uint8_t transparency;

  TransparentShape(Shape& shape, const uint8_t transparency)
    : shape{shape}, transparency{transparency} {}

  string str() const override
  {
    ostringstream oss;
    oss << shape.str() << " has "
      << static_cast<float>(transparency) / 255.f*100.f
      << "% transparency";
    return oss.str();
  }
};
```

我们现在有一个增强功能，它接收一个 0 到 255 范围内的透明度值，并将其报告为百分比值。我们不能单独使用这个增强功能：

```c++
Square square{3};
TransparentShape demiSquare{square, 85};
cout << demiSquare.str();
// A square with side 3 has 33.333% transparency
```

但很棒的一点是，我们可以将 `ColoredShape` 和 `TransparentShape` 组合在一起，以创建一个既有颜色又有透明度的形状：

```c++
TransparentShape myCircle{
  ColoredShape{
    Circle{23}, "green"
  }, 64
};
cout << myCircle.str();
 // A circle of radius 23 has the color green has 25.098% transparency
```

您看，我在这里做了什么？我只是在原地构建了整个对象。现在，为了公平起见，有一件事虽然可以做但并没有太多意义，那就是重复使用同一个装饰器。例如，拥有一个 `ColoredShape{ColoredShape{...}}` 并没有实际意义，但它确实会工作，并给出有些矛盾的结果。如果您决定通过断言或某些面向对象的技巧来对抗这种情况——当然，您可以这样做，但我很好奇您将如何处理类似的情况：

```c++
ColoredShape{TransparentShape{ColoredShape{...}}}
```

这种情况更难以检测，即使它是可能的，我认为空间检查是否值得。我们需要假设程序员会保持一定的理智。

## Static Decorator

您是否注意到，在设置场景时，我给 `Circle` 添加了一个叫做 `resize()` 的函数，而它并不是 `Shape` 接口的一部分？正如您可能猜到的，由于它不是 `Shape` 接口的一部分，您确实无法从装饰器中调用它。我的意思是：

```c++
Circle circle{3};
ColoredShape redCircle{circle, "red"};
redCircle.resize(2); // won't compile!
```

因此，假设您并不关心是否可以在运行时组合对象，但您确实关心能够访问所有被装饰对象的字段和成员函数。是否有可能构造这样的装饰器呢？

实际上，这是可行的，并且是通过模板和继承来实现的——但不是那种会导致状态空间爆炸的继承。相反，我们将应用一种称为“混入继承”（Mixin Inheritance）的技术，即类从其自身的模板参数继承。

所以，我们的想法是创建一个新的 `ColoredShape`，它从一个模板参数继承。我们没有办法限制模板参数必须是某种特定类型，因此我们将使用 `static_assert` 来进行类型检查：

```c++
template <typename T>
struct ColoredShape : T
{
  static_assert(is_base_of<Shape, T>::value,
    "Template argument must be a Shape");

  string color;
  string str() const override
  {
    ostringstream oss;
    oss << T::str() << " has the color " << color;
    return oss.str();
  }
}; // implementation of TransparentShape<T> omitted
```

借助 `ColoredShape<T>` 和 `TransparentShape<T>` 的实现，我们现在可以将它们组合成一个既有颜色又有透明度的形状：

```c++
ColoredShape<TransparentShape<Square>> square{"blue"};
square.size = 2;
square.transparency = 0.5;
cout << square.str();
// can call square's own members
square.resize(3);
```

这难道不是很好吗？当然，好但并不完美：我们似乎失去了对构造函数的完全使用，所以即使我们能够初始化最外层的类，也无法在一行代码中完全构造出具有特定大小、颜色和透明度的形状。

为了给我们的“蛋糕”加上最后一层糖霜（装饰！），让我们为 `ColoredShape` 和 `TransparentShape` 添加转发构造函数。这些构造函数将接受两个参数：第一个是当前模板类特有的参数，第二个是一个通用的参数包，我们将这个参数包转发给基类。我的意思是：

```c++
template <typename T>
struct TransparentShape : T
{
  uint8_t transparency;

  template<typename...Args>
  TransparentShape(const uint8_t transparency, Args ...args)
    : T(std::forward<Args>(args)...)
    , transparency{ transparency } {}
  ...
}; // same for ColoredShape
```

为了重申一下，前述构造函数可以接受任意数量的参数，其中第一个参数用于初始化透明度值，而其余参数则简单地转发给基类的构造函数，无论那个基类具体是什么。

构造函数的数量自然必须是正确的，如果构造函数的数量或参数类型不正确，程序将无法编译。如果您开始为类型添加默认构造函数，整体参数集的使用会变得更加灵活，但也可能会引入歧义和混淆。

另外，请确保不要将这些构造函数声明为 `explicit`，否则在组合装饰器时会违反 C++ 的 copy-list-initialization（复制列表初始化）规则。现在，如何实际使用所有这些功能呢？

```c++
ColoredShape2<TransparentShape2<Square>> sq = { "red", 51, 5 };
cout << sq.str() << endl;
// A square with side 5 has 20% transparency has the color red
```

太棒了！这正是我们所希望的。这完成了我们静态装饰器的实现。再次说明，您可以通过各种模板技巧来增强它，以避免重复的类型如 `ColoredShape<ColoredShape<…>>` 或循环类型如 `ColoredShape<TransparentShape<ColoredShape<...>>>`，但在静态上下文中，这样做感觉像是浪费时间。尽管如此，这些确实是完全可以实现的。

## Functional Decorator

虽然装饰器模式通常应用于类，但它同样可以应用于函数。例如，假设代码中的某个特定操作给您带来了麻烦：您希望记录每次调用该操作的情况，并在 Excel 中分析统计数据。这当然可以通过在调用前后添加一些代码来实现，即：

```c++
cout << "Entering function\n";
// do the work
cout << "Exiting funcion\n";
```

这种方法确实可行，但在关注点分离方面并不理想：我们真正希望将日志记录功能存储在某个地方，以便我们可以根据需要重用和增强它。

有不同的方法可以实现这一点。一种方法是将整个工作单元作为 lambda 表达式传递给类似于以下的某个日志组件：

```c++
struct Logger
{
  function<void()> func;
  string name;

  Logger(const function<void()>& func, const string& name)
    : func{func},
      name{name}
  {
  }

  void operator()() const
  {
    cout << "Entering " << name << endl;
    func();
    cout << "Exiting " << name << endl;
  }
};
```

采用这种方法，您可以编写如下内容：

```c++
Logger([]() {cout << "Hello" << endl; }, "HelloFunction")();
// output:
// Entering HelloFunction
// Hello
// Exiting HelloFunction
```

始终有一个选项是不将函数作为 `std::function` 传递，而是作为模板参数传递。这会导致前述方法的 `slight` 变体：

```c++
template <typename Func>
struct Logger2
{
  Func func;
  string name;

  Logger2(const Func& func, const string& name)
    : func{func}, name{name} {}
  
  void operator()() const
  {
    cout << "Entering " << name << endl;
    func();
    cout << "Exiting " << name << endl;
  }
};
```

前述实现的用法是完全相同的。我们可以创建一个实用函数来实际生成这样的日志记录器：

```c++
template <typename Func>
auto make_logger2(Func func, const string& name)
{
  return Logger2<Func>{ func, name }; // () = call now
}
```

然后可以像这样使用它：

```c++
auto call = make_logger2([]() {cout << "Hello!" << endl; }, "HelloFunction");
call();
```

“这有什么意义呢？”您可能会问。嗯……我们现在有了创建装饰器的能力（将被装饰的函数包含在内），并且可以在我们选择的时间调用它。

现在，这里有一个挑战给您：如果您想记录函数 `add()` 的调用，该函数定义如下……

```c++
double add(double a, double b)
{
  cout << a << "+" << b << "=" << (a + b) << endl;
  return a + b;
}
```

但如果您还想获取返回值呢？是的，从日志记录器返回一个返回值。这并不容易！但当然也不是不可能。让我们创建日志记录器的又一个版本：

```c++
template <typename R, typename... Args>
struct Logger3<R(Args...)>
{
  Logger3(function<R(Args...)> func, const string& name)
    : func{func},
      name{name}
  {
  }

  R operator() (Args ...args)
  {
    cout << "Entering " << name << endl;
    R result = func(args...);
    cout << "Exiting " << name << endl;
    return result;
  }

  function<R(Args ...)> func;
  string name;
};
```

在前述内容中，模板参数 `R` 指的是返回值的类型，而 `Args`，您无疑已经猜到了。装饰器保存函数并在必要时调用它，唯一的区别是 `operator()` 返回一个 `R`，因此不会丢失返回值。

我们可以构建另一个实用的 `make_` 函数：

```c++
template <typename R, typename... Args>
auto make_logger3(R (*func)(Args...), const string& name)
{
  return Logger3<R(Args...)>(
    std::function<R(Args...)>(func),
    name);
}
```

请注意，我没有使用 `std::function`，而是将第一个参数定义为普通的函数指针。我们现在可以使用这个函数来实例化带日志的调用并使用它：

```c++
auto logged_add = make_logger3(add, "Add");
auto result = logged_add(2, 3);
```

当然，`make_logger3` 可以用依赖注入（Dependency Injection）来替代。这种方法的优点是能够：

- 动态地通过提供一个空对象（Null Object）而不是实际的日志记录器来开关日志记录功能
- 通过替换为不同的日志记录器来禁用被记录代码的实际调用

总而言之，这是开发者的工具箱中的另一个有用工具。我将这种做法融入依赖注入的具体实现留给读者作为练习。

## 总结

装饰器模式为类提供额外功能的同时遵循开闭原则（OCP）。其关键方面是可组合性：多个装饰器可以按任意顺序应用于一个对象。我们已经讨论了以下几种类型的装饰器：

- *动态装饰器*可以存储被装饰对象的引用（或者如果您愿意，甚至可以存储整个值），并提供动态（运行时）组合能力，但代价是无法访问底层对象自身的成员。
- *静态装饰器*使用混入继承（从模板参数继承）在编译时组合装饰器。这失去了任何运行时的灵活性（您不能重新组合对象），但允许访问底层对象的成员。这些对象也可以通过构造函数转发完全初始化。
- *函数式装饰器*可以包装代码块或特定函数，以允许行为的组合。

值得一提的是，在不允许多重继承的语言中，装饰器还用于通过聚合多个对象并提供这些聚合对象接口的集合联合来模拟多重继承。
