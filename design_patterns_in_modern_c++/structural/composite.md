# Composite

确实，对象经常由其他对象组成（或者换句话说，它们聚合了其他对象）。请记住，在这部分书的开始，我们同意将聚合和组合视为等同。

对象表明它由其他东西组成的办法非常有限。字段本身并不构成接口，除非您创建了虚拟的 `getter` 和 `setter` 方法。您可以通过实现 `begin()` / `end()` 成员来宣传自己是由对象集合组成的，但请注意，这实际上并没有说明很多：毕竟，您可以在这些方法中做任何您想做的事情。同样地，您可以通过定义迭代器类型来暗示自己是某种特定类型的容器，但真的有人会去检查吗？

另一种宣传自己是容器的方法是从容器继承。这通常是可以接受的：即使 STL 容器没有虚析构函数，如果您在自己的析构函数中也不需要做什么，并且不期望有人会从您的类型继承，那么一切都没问题——您可以继承自 `std::vector`，不应该发生什么坏事。

那么，组合模式（Composite pattern）是关于什么呢？本质上，我们尝试给单个对象和对象组提供一个相同的接口。当然，简单地定义一个接口并在两者中实现它是很容易的。同样地，您也可以尝试在适用的情况下利用如 `begin()` / `end()` 这样的鸭子类型机制。然而，一般来说，鸭子类型是一个糟糕的想法，因为它依赖于隐秘的知识而不是在某个接口中明确定义。顺便说一句，没有什么可以阻止您创建带有 `begin()` 和 `end()` 的显式接口，但是迭代器类型是什么呢？

## Array Backed Properties

组合设计模式通常应用于整个类，但在我们深入这一点之前，我想向您展示它如何在属性的层面上使用。这里所说的属性，当然指的是类的字段以及这些字段如何暴露给 API 使用者。

想象一下一个电脑游戏，其中的生物具有不同的数值特征。每个生物可以有一个力量值、一个敏捷值等。因此，定义这些是非常简单的：

```c++
class Creature
{
  int strength, agility, intelligence;
public:
  int get_strength() const;
  {
    return strength;
  }

  void set_strength(int strength)
  {
    Creature::strength = strength;
  }

  // other getter and setters here
};
```

到目前为止，一切顺利。但现在想象一下，我们想要计算生物的一些聚合统计数据。例如，我们想知道其所有统计值的总和、所有统计值的平均值以及，比如说，最高值。由于我们的数据分散在各个字段中，我们最终会得到如下实现：

```c++
class Creature
{
  // other members here
  int sum() const {
    return strength + agility + intelligence;
  }

  double average() const {
    return sum() / 3.0;
  }

  int max() const {
    return ::max(::max(strength, agility), intelligence);
  }
};
```

这种实现方式由于以下几个原因而不令人满意：

- 在计算所有统计数据的总和时，我们很容易犯错并忘记其中一个统计值。
- 在计算平均值时，我们使用了一个确实的魔数 `3.0`，它对应于参与计算的字段数量。
- 在计算最大值时，我们必须构建 `std::max()` 调用的成对组合。

现有的代码已经很糟糕了；现在想象一下再添加一个属性。这将需要对 `sum()`、`average()`、`max()` 以及任何其他聚合计算进行真正糟糕的重构。这种情况可以避免吗？事实证明是可以的。

使用数组支持的属性的方法如下。首先，我们为所有必需的属性定义枚举成员，然后创建适当大小的数组：

```c++
class Creature
{
  enum Abilities { str, agl, intl, count };
  array<int, count> abilities;
};
```

上述枚举定义包含一个额外的值 `count`，它告诉我们总共有多少个元素。请注意，我们使用的是普通的 `enum`，而不是 `enum class`，这使得这些成员的使用稍微容易一些。

我们现在可以为力量、敏捷性等定义 `getter` 和 `setter`，它们将被映射到我们的后端数组中，例如：

```c++
int get_strength() const { return abilities[str]; }
void set_strength(int value) { abilities[str] = value; }
// same for other properties
```

这种代码是 IDE 不会为您生成的，但为了灵活性付出这一点点代价是值得的。

现在，精彩的来了：我们的 `sum()`、`average()` 和 `max()` 计算变得非常简单，因为在所有这些情况下，我们所要做的只是遍历一个数组：

```c++
int sum() const {
  return accumulate(abilities.begin(), abilities.end(), 0);
}

double average() const {
  return sum() / (double)count;
}

int max() const {
  return *max_element(abilities.begin(), abilities.end());
}
```

这难道不是很好吗？不仅代码编写和维护变得更加简单，而且向类中添加一个新属性也只需新增一个枚举成员和一对 `getter-setter`；聚合函数完全不需要改动！

## Grouping Graphic Objects

想象一下像 PowerPoint 这样的应用程序，您可以在其中选择多个不同的对象并将它们作为一个整体拖动。然而，如果您选择单个对象，也可以单独移动那个对象。渲染也是一样：您可以渲染单个图形对象，或者可以将多个形状组合在一起作为一组绘制。

这种做法的实现相当简单，因为它依赖于一个单一的接口，例如以下：

```c++
struct GraphicObject
{
  virtual void draw() = 0;
};
```

从名称上看，您可能会认为 `GraphicObject` 总是单一的，也就是说，它总是表示一个单独的项目。然而，请仔细考虑：多个矩形和圆形组合在一起代表一个复合图形对象（因此得名组合设计模式）。所以，就像我可以定义一个圆一样：

```c++
struct Circle : GraphicObject
{
  void draw() override
  {
    std::cout << "Circle" << std::endl;
  }
};
```

同理，我可以定义一个由多个其他图形对象组成的 `GraphicObject`。是的，这种关系可以无限递归：

```c++
struct Group : GraphicObject
{
  std::string name;

  explicit Group(const std::string& name)
    : name{name} {}

  void draw() override
  {
    std::cout << "Group " << name.c_str() << " contains:" << std::endl;
    for (auto&& o : objects)
      o->draw();
  }

  std::vector<GraphicObject*> objects;
};
```

无论是单一的 `Circle` 还是任何 `Group`，只要它们都实现了 `draw()` 函数，就是可绘制的。`Group` 维护一个指向其他图形对象的指针向量（这些对象也可以是 `Groups` ！），并使用该向量来渲染自身。

这就是如何使用此 API 的方式：

```c++
Group root("root");
Circle c1, c2;
root.objects.push_back(&c1);

Group subgroup("sub");
subgroup.objects.push_back(&c2);

root.objects.push_back(&subgroup);

root.draw();
```

前述代码生成以下输出：

```
Group root contains:
Circle
Group sub contains:
Circle
```

这就是组合设计模式的最简单实现，尽管使用的是我们自己定义的自定义接口。现在，如果我们尝试采用一些其他更为标准化的遍历对象的方法，这个模式会是什么样子呢？

## Neural Networks

机器学习是当前的热门领域，我希望它能保持这种热度，否则我将不得不更新这一段落。机器学习的一部分内容是使用人工神经网络：这些软件构造试图模仿我们大脑中神经元的工作方式。

神经网络的核心概念当然是神经元。一个神经元可以根据其输入产生一个（通常是数值的）输出，并且我们可以将该值传递给网络中的其他连接。我们将只关注连接部分，因此我们将这样建模神经元：

```c++
struct Neuron
{
  vector<Neuron*> in, out;
  unsigned int id;

  Neuron()
  {
    static int id = 1;
    this->id = id++;
  }
};
```

我添加了 id 字段用于识别。现在，您可能想要做的是能够将一个神经元连接到另一个神经元，这可以通过以下方式实现：

```c++
template <>
void connect_to<Neuron>(Neuron& other)
{
  out.push_back(&other);
  other.in.push_back(this);
}
```

这个函数做的事情相当可以预见：它在当前（this）神经元和另一个神经元之间建立连接。到目前为止，一切顺利。

现在，假设我们还想创建神经元层。一层很简单，就是特定数量的神经元组合在一起。让我们再次犯下从 `std::vector` 继承这一大忌：

```c++
struct NeuronLayer : vector<Neuron>
{
  NeuronLayer(int count)
  {
    while (count --> 0)
      emplace_back(Neuron{});
  }
};
```

看起来不错，对吧？我甚至加入了 `operator-->` 让您享受。但现在，我们遇到了一点问题。

问题是这样的：我们希望神经元能够连接到神经元层。大致来说，我们希望以下情况能够工作：

```c++
Neuron n1, n2;
NeuronLayer layer1, layer2;
n1.connect_to(n2);
n1.connect_to(layer1);
layer1.connect_to(n1);
layer1.connect_to(layer2);
```

如您所见，我们有四种不同的情况需要处理：

1. 神经元连接到另一个神经元
2. 神经元连接到层
3. 层连接到神经元；以及
4. 层连接到另一层

如您可能已经猜到的，我们不可能为 `connect_to()` 成员函数创建四个重载版本。如果存在三个不同的类——您会实际考虑创建九个函数吗？我想不会。

相反，我们将要做的是插入一个基类——多亏了多重继承，我们完全可以这样做。那么，如下所示如何？

```c++
template <typename Self>
struct SomeNeurons
{
  template <typename T> void connect_to(T&  other)
  {
    for (Neuron& from : *static_cast<Self*>(this))
    {
      for (Neuron& to : other)
      {
        from.out.push_back(&to);
        to.in.push_back(&from);
      }
    }
  }
};
```

`connect_to()` 的实现绝对值得讨论。如您所见，它是一个模板成员函数，接受类型 `T`，然后继续成对迭代 `*this` 和 `T&` 的神经元，逐对连接每个神经元。但有一个需要注意的地方：我们不能直接迭代 `*this`，因为这将给我们一个 `SomeNeurons&`，而我们真正需要的是实际类型。

这就是为什么我们必须使 `SomeNeurons&` 成为一个模板类，其中模板参数 `Self` 指的是继承者类。然后我们在解引用并迭代内容之前，将 `this` 指针强制转换为 `Self*`。这意味着 `Neuron` 必须从 `SomeNeurons<Neuron>` 继承——为了这种便利性，这是一个小小的成本。

剩下的就是分别在 `Neuron和NeuronLayer` 中实现 `SomeNeurons::begin()` 和 `end()`，以便基于范围的 for 循环能够正常工作。

由于 `NeuronLayer` 继承自 `vector<Neuron>`，因此显式实现 `begin()` / `end()` 对是不必要的——它们已经自动存在了。但是 `Neuron` 确实需要一种方式来迭代……实际上是它自己。它需要把自己作为唯一可迭代元素返回。这可以通过以下方式完成：

```c++
Neuron* begin() override { return this; }
Neuron* end() override { return this + 1; }
```

我给您一点时间来欣赏这个设计的巧妙之处。正是这段“魔法”使得 `SomeNeurons::connect_to()` 成为可能。简而言之，我们让一个单一（标量）对象表现得像一个可迭代的对象集合。这允许以下所有用法：

```c++
Neuron neuron, neuron2;
NeuronLayer layer, layer2;

neuron.connect_to(neuron2);
neuron.connect_to(layer);
layer.connect_to(neuron);
layer.connect_to(layer2);
```

更不用说，如果您引入了一个新的容器（比如说某种 `NeuronRing`），您所需要做的只是从 `SomeNeurons<NeuronRing>` 继承，并实现 `begin()` / `end()`，新的类将立即可以与所有的 `Neuron` 和 `NeuronLayer` 连接。

## 总结

组合设计模式允许我们为单个对象和对象集合提供相同的接口。这可以通过显式使用接口成员来实现，或者通过鸭子类型来实现——例如，基于范围的 for 循环不要求您继承任何东西，而是基于类型具有看起来合适的 `begin()` / `end()`成员。

正是这些 `begin()` / `end()` 成员使得标量类型可以伪装成“集合”。值得注意的是，我们的 `connect_to()` 函数中的嵌套 for 循环能够将两种构造连接在一起，即使它们具有不同的迭代器类型：`Neuron` 返回 `Neuron*`，而 `NeuronLayer` 返回 `vector<Neuron>::iterator`——这两者并不完全相同。啊，模板的魔力！

最后，我必须承认，所有这些复杂的处理只在您希望拥有一个成员函数时是必要的。如果您愿意调用全局函数，或者您对拥有多个 `connect_to()` 实现感到满意，那么基类 `SomeNeurons` 就不是必需的。
