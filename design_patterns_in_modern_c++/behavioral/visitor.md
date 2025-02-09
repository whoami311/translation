# Visitor

一旦你有了类型层次结构，除非你有访问源代码的权限，否则不可能向层次结构的每个成员添加函数。这是一个需要提前规划的问题，并由此产生了访问者（Visitor）模式。

这里有一个简单的例子：假设你已经解析了一个数学表达式（当然，使用了解释器（Interpreter）模式！），该表达式由 `double` 值和加法运算符组成：

```c++
(1.0 + (2.0 + 3.0))
```

这个表达式可以表示为类似于以下的层次结构：

```c++
struct Expression
{
  // nothing here (yet)
};

struct DoubleExpression : Expression
{
  double value;
  explicit DoubleExpression(const double value)
    : value{value} {}
};

struct AdditionExpression : Expression
{
  Expression *left, *right;

  A dditionExpression(Expression* const left, Expression* const right)
    : left{left}, right{right} {}

  ~AdditionExpression()
  {
    delete left; delete right;
  }
};
```

因此，鉴于这个对象的层级结构，假设您想要为各个 `Expression` 继承者添加一些行为（好吧，我们现在只有两个，但这个数量可以增加）。您会怎么做？

##  Intrusive Visitor

我们将从最直接的方法开始，这种方法会违背开放封闭原则（Open-Closed Principle）。实际上，我们将进入已经编写好的代码，并修改 `Expression` 接口（以及由此关联的每一个派生类）：

```c++
struct Expression
{
  virtual void print(ostringstream& oss) = 0;
};
```

除了违背 OCP 之外，这种修改还依赖于您实际上有权访问该层级结构的源代码这一假设——而这并不是总能保证的。但我们总得从某个地方开始，对吧？因此，通过这次修改，我们需要在 `DoubleExpression` 中实现 `print()` 方法（这很简单，所以这里我将省略）以及在 `AdditionExpression` 中也实现它：

```c++
struct AdditionExpression : Expression
{
  Expression *left, *right;
  ...
  void print(ostringstream& oss) override
  {
    oss << "(";
    left->print(oss);
    oss << "+";
    right->print(oss);
    oss << ")";
  }
};
```

哇，这很有趣！我们正在多态且递归地调用子表达式的 `print()`。太棒了；让我们来测试一下：

```c++
auto e = new AdditionExpression{
  new DoubleExpression{1},
  new AdditionExpression{
    new DoubleExpression{2},
    new DoubleExpression{3}
  }
};
ostringstream oss;
e->print(oss);
cout << oss.str() << endl; // prints (1+(2+3))
```

好吧，这很简单。但现在想象一下，如果在层级结构中有十个继承者（实际上，在现实场景中这并不罕见），而您需要添加一个新的 `eval()` 操作。这就意味着需要在十个不同的类中进行十次修改。但 OCP 并不是真正的问题。

真正的问题是 SRP。您看，像打印这样的问题是一个特殊的关注点。与其声明每个表达式都应该能够打印自己，为什么不引入一个 `ExpressionPrinter`，让它知道如何打印表达式呢？而且，之后您可以引入一个 `ExpressionEvaluator`，让它知道如何执行实际的计算，这一切都不会以任何方式影响 `Expression` 的层级结构。

## Reflective Printer

现在我们已经决定创建一个独立的打印组件，让我们去掉 `print()` 成员函数（但当然要保留基类）。这里有一个需要注意的地方：您不能让 `Expression` 类为空。为什么？因为只有当类中确实有 `virtual` 成员时，才能获得多态行为。所以，目前，让我们在其中加入一个虚析构函数；这就够了！

```c++
struct Expression
{
  virtual ~Expression() = default;
};
```

现在让我们尝试实现一个 `ExpressionPrinter`。我的第一直觉是写出类似这样的代码：

```c++
struct ExpressionPrinter
{
  void print(DoubleExpression *de, ostringstream& oss) const
  {
    oss << de->value;
  }

  void print(AdditionExpression *ae, ostringstream& oss) const
  {
    oss << "(";
    print(ae->left, oss);
    oss << "+";
    print(ae->right, oss);
    oss << ")";
  }
};
```

前面的代码编译成功的几率：零。C++ 知道，比如说，`ae->left` 是一个 `Expression`，但由于它不在运行时检查类型（不像一些动态类型语言），因此它不知道应该调用哪个重载函数。真遗憾！

这里可以做什么呢？只有一种方法——移除重载并在运行时检查类型：

```c++
struct ExpressionPrinter
{
  void print(Expression *e)
  {
    if (auto de = dynamic_cast<DoubleExpression*>(e))
    {
      oss << de->value;
    }
    else if (auto ae = dynamic_cast<AdditionExpression*>(e))
    {
      oss << "(";
      print(ae->left, oss);
      oss << "+";
      print(ae->right, oss);
      oss << ")";
    }
  }

  string str() const { return oss.str(); }
private:
  ostringstream oss;
};
```

前述的实际上是一个可用的解决方案：

```c++
auto e = new AdditionExpression{
  new DoubleExpression{ 1 },
  new AdditionExpression{
    new DoubleExpression{ 2 },
    new DoubleExpression{ 3 }
  }
};
ExpressionPrinter ep;
ep.print(e);
cout << ep.str() << endl; // prints "(1+(2+3))"
```

这种方法有一个相当明显的缺点：编译器无法检查您是否确实为层级结构中的每个元素都实现了打印功能。

当添加一个新的元素时，您可以继续使用 `ExpressionPrinter` 而无需修改，它将直接跳过任何新类型元素。

但这仍然是一种可行的解决方案。认真地说，可以就此打住而不必进一步深入访问者（Visitor）模式：`dynamic_cast` 的开销并不是很大，并且我认为许多开发人员会记得在该 `if` 语句中覆盖每种对象类型。

## WTH is Dispatch?

每当人们谈论访问者模式时，都会提到 dispatch（分派）这个词。它是什么？简单来说，“dispatch” 是指确定调用哪个函数的问题——具体而言，需要多少信息来决定进行调用。

这里有一个简单的例子：

```c++
struct Stuff {}
struct Foo : Stuff {}
struct Bar : Stuff {}

void func(Foo* foo) {}
void func(Bar* bar) {}
```

现在，如果我创建一个普通的 `Foo` 对象，使用它调用 `func()` 将不会有任何问题：

```c++
Foo *foo = new Foo;
func(foo); // ok
```

但是，如果我决定将其转换为基类指针，那么编译器将不知道应该调用哪个重载函数：

```c++
Stuff *stuff = new Foo;
func(stuff); // oops!
```

现在，让我们从多态性的角度思考：有没有办法可以在不进行任何运行时（如 `dynamic_cast` 等）检查的情况下，迫使系统调用正确的重载函数？事实证明这是可行的。

您看，当您在 `Stuff` 上调用某个方法时，这个调用可以是多态的（得益于虚函数表 Vtable），并且它可以被直接分派到必要的组件。这反过来又可以调用所需的重载函数。这被称为 *double dispatch*（双重分派），因为：

1. 首先，您对实际对象执行一个多态调用。
2. 在多态调用的内部，您再调用重载函数。由于在对象内部，`this` 的类型是确切的（例如，一个 `Foo*` 或 `Bar*`），因此会触发正确的重载函数。

以下是我所指的具体实现方式：

```c++
struct Stuff {
  virtual void call() = 0;
}

struct Foo : Stuff {
  void call() override { func(this); }
}

struct Bar : Stuff {
  void call() override { func(this); }
}

void func(Foo* foo) {}
void func(Bar* bar) {}
```

您能看出来这里发生了什么吗？我们不能仅仅在 `Stuff` 中放入一个通用的 `call()` 实现：不同的实现必须位于各自的类中，以便 `this` 指针能够正确地被类型化。

这种实现方式让您能够编写如下代码：

```c++
Stuff *stuff = new Foo;
stuff->call(); // effectively calls func(stuff);
```

## Classic Visitor

“经典”的 Visitor 设计模式实现使用了双重分派。关于访问者成员函数的命名有一些约定：

- 访问者的成员函数通常被称为 `visit()`。
- 在层级结构中实现的成员函数通常被称为 `accept()`。

现在我们可以去掉 `Expression` 基类中的那个虚析构函数，因为我们实际上有东西可以放在那里：`accept()` 函数：

```c++
struct Expression
{
  virtual void accept(ExpressionVisitor *visitor) = 0;
};
```

如您所见，前述内容指的是一个名为 `ExpressionVisitor` 的（抽象）类，它可以作为各种访问者的基类，例如 `ExpressionPrinter`、`ExpressionEvaluator` 等。我在这里选择使用指针，但您也可以使用引用。

现在，每个 `Expression` 的继承者都必须以相同的方式实现 `accept()`，即：

```c++
void accept(ExpressionVisitor* visitor) override
{
  visitor->visit(this);
}
```

另一方面，我们可以如下定义 `ExpressionVisitor`：

```c++
struct ExpressionVisitor
{
  virtual void visit(DoubleExpression* de) = 0;
  virtual void visit(AdditionExpression* ae) = 0;
};
```

请注意，我们必须为所有对象定义重载；否则，在实现相应的 `accept()` 时我们会遇到编译错误。现在我们可以从这个类继承来定义我们的 `ExpressionPrinter`：

```c++
struct ExpressionPrinter : ExpressionVisitor
{
  ostringstream oss;
  string str() const { return oss.str(); }
  void visit(DoubleExpression* de) override;
  void visit(AdditionExpression* ae) override;
};
```

`visit()` 函数的实现应该是相当明显的，因为我们已经看过不止一次了，但我会再展示一次：

```c++
void ExpressionPrinter::visit(AdditionExpression* ae)
{
  oss << "(";
  ae->left->accept(this);
  oss << "+";
  ae->right->accept(this);
  oss << ")";
}
```

请注意，调用现在是发生在子表达式本身上，再次利用了双重分派。至于新的双重分派 Visitor 的使用方式，如下所示：

```c++
void main()
{
  auto e = new AdditionExpression{
    // as before
  };
  ostringstream oss;
  ExpressionPrinter ep;
  ep.visit(e);
  cout << ep.str() << endl; // (1+(2+3))
}
```

### Implementing an Additional Visitor

那么，这种方法的优势是什么呢？优势在于您只需在整个层级结构中实现一次 `accept()` 成员函数。之后您将不再需要触层级结构中的任何成员。例如：假设现在您想要有一种方法来评估表达式的结果，这很简单：

```c++
struct ExpressionEvaluator : ExpressionVisitor
{
  double result;
  void visit(DoubleExpression* de) override;
  void visit(AdditionExpression* ae) override;
};
```

但需要注意的是，`visit()` 当前被声明为一个 `void` 方法，因此实现可能会看起来有点奇怪：

```c++
void ExpressionEvaluator::visit(DoubleExpression* de)
{
  result = de->value;
}

void ExpressionEvaluator::visit(AdditionExpression* ae)
{
  ae->left->accept(this);
  auto temp = result;
  ae->right->accept(this);
  result += temp;
}
```

前述内容是由于无法从 `accept()` 返回值所导致的，实现起来稍微有些复杂。实际上，我们评估左部分并缓存该值。然后评估右部分（这样 `result` 就被设置了），接着用我们缓存的值增加它，从而产生总和。这并不是非常直观的代码！

尽管如此，它工作得很好：

```c++
auto e = new AdditionExpression{ /* as before */ };
ExpressionPrinter printer;
ExpressionEvaluator evaluator;
printer.visit(e);
evaluator.visit(e);
cout << printer.str() << " = " << evaluator.result << endl;
// prints "(1+(2+3)) = 6"
```

同样地，您可以添加许多其他不同的访问者，在遵循 OCP 的同时享受这个过程。

## Acyclic Visitor

现在是时候提到，实际上，访问者设计模式存在两种变体，如果您愿意这么说的话。它们是：

- **Cyclic Visitor（循环访问者）**，它基于函数重载。由于层级结构（必须知道访问者的类型）和访问者（必须知道层级结构中的每一个类）之间的循环依赖，这种方法的使用被限制在稳定且不常更新的层级结构上。
- **Acyclic Visitor（非循环访问者）**，它基于 RTTI（运行时类型信息）。这里的优势是访问的层级结构没有限制，但如您可能已经猜到的，这会有性能方面的影响。

在实现 Acyclic Visitor 的第一步是定义实际的访问者接口。我们不是为层级结构中的每个类型都定义一个 `visit()` 重载，而是尽可能地使事情变得通用：

```c++
template <typename Visitable>
struct Visitor
{
  virtual void visit(Visitable& obj) = 0;
};
```

我们需要域模型中的每个元素都能够接受这样的访问者，但由于每个特化都是独特的，我们引入了一个 *marker interface*（标记接口）——一个除了虚析构函数外没有任何内容的空类：

```c++
struct VisitorBase // marker interface
{
  virtual ~VisitorBase() = default;
};
```

前述的类没有行为，但我们将使用它作为 `accept()` 方法的参数，应用于我们实际想要访问的任何对象。现在，我们可以如下重新定义之前的 `Expression` 类：

```c++
struct Expression
{
  virtual ~Expression() = default;

  virtual void accept(VisitorBase& obj)
  {
    using EV = Visitor<Expression>;
    if (auto ev = dynamic_cast<EV*>(&obj))
      ev->visit(*this);
  }
};
```

新的 `accept()` 方法的工作方式如下：我们接受一个 `VisitorBase`，但随后尝试将其转换为 `Visitor<T>`，其中 `T` 是我们当前所在的类型。如果转换成功，则表示该访问者知道如何访问我们的类型，因此我们调用它的 `visit()` 方法。如果失败，则不执行任何操作（no-op）。理解为什么 `obj` 本身没有我们可以直接调用的 `visit()` 方法是关键的。如果它有，则需要为每一个可能调用它的类型提供重载，这正是引入循环依赖的原因。

在模型的其他部分实现了 `accept()` 之后，我们可以通过再次定义 `ExpressionPrinter` 来将所有内容整合在一起，但这一次它的定义如下：

```c++
struct ExpressionPrinter : VisitorBase,
                           Visitor<DoubleExpression>,
                           Visitor<AdditionExpression>
{
  void visit(DoubleExpression &obj) override;
  void visit(AdditionExpression &obj) override;
  string str() const { return oss.str(); }
private:
  ostringstream oss;
};
```

如您所见，我们不仅实现了 `VisitorBase` 标记接口，还为每个我们想要访问的类型 `T` 实现了 `Visitor<T>`。如果我们省略了某个特定类型 `T`（例如，假设我注释掉了 `Visitor<DoubleExpression>`），程序仍然可以编译，如果发生对应的 `accept()` 调用，它将简单地执行为空操作（no-op）。

在前述内容中，`visit()` 方法的实现几乎与经典访问者实现中的相同，结果也是如此。

## Variants and std::visit

虽然与经典的 Visitor 模式没有直接关系，但值得一提的是 `std::visit`，至少因为它的名字暗示了与 Visitor 模式有关。实际上，`std::visit` 是访问变体类型（variant type）正确部分的一种方式。

这里有一个例子：假设您有一个地址，其中一部分是 `house` 字段。现在，房屋可以只是一个数字（如“123 London Road”），或者它可以有一个名称，例如“Montefiore Castle”。因此，您可以如下定义变体：

```c++
variant<string, int> house;
// house = "Montefiore Castle";
house = 221;
```

这两种赋值都是有效的。现在，假设您决定打印房屋名称或编号。为此，您可以首先定义一个类型，该类型对变体内部的不同成员类型具有函数调用重载：

```c++
struct AddressPrinter
{
  void operator()(const string& house_name) const {
    cout << "A house called " << house_name << "\n";
  }

  void operator()(const int house_number) const {
    cout << "House number " << house_number << "\n";
  }
};
```

现在，这种类型可以与 `std::visit()` 结合使用，`std::visit()` 是一个库函数，它将此访问者应用到变体类型上：

```c++
AddressPrinter ap;
std::visit(ap, house); // House number 221
```

得益于一些现代 C++ 的特性，我们也可以就地定义访问者函数集。我们需要做的是构造一个类型为 `auto&` 的 lambda 表达式，获取底层类型，使用 `if constexpr` 进行比较，并相应地处理：

```c++
std::visit([](auto& arg) {
  using T = decay_t<decltype(arg)>;

  if constexpr (is_same_v<T, string>)
  {
    cout << "A house called " << arg.c_str() << "\n";
  }
  else
  {
    cout << "House number " << arg << "\n";
  }
}, house);
```

## 总结

访问者设计模式允许我们为对象层级结构中的每个元素添加一些行为。我们已经看到的方法包括：

- *侵入式*：向层级结构中的每个对象添加一个虚方法。这是可能的（假设您有权访问源代码），但它违反了开放封闭原则（OCP）。
- *反射式*：添加一个独立的访问者，该访问者不需要对对象进行任何修改；在需要运行时分派时使用 `dynamic_cast`。
- *经典（双重分派）*：整个层级结构确实会得到修改，但只是一次，并且是以非常通用的方式。层级结构的每个元素学会 `accept()` 一个访问者。然后我们通过继承访问者来增强层级结构的功能，向各种方向扩展。

访问者模式经常与解释器模式一起使用：在解释了一些文本输入并将其转换为面向对象的结构之后，我们需要例如以特定方式渲染抽象语法树。访问者有助于在整个层级结构中传播 `ostringstream`（或类似对象），并将数据汇总在一起。
