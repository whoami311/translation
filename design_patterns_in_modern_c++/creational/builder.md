# Builder

构建器（Builder）模式关注的是复杂对象的创建，即那些无法通过单行构造函数调用就能构建的对象。这类对象可能由其他对象组成，并且其构建过程可能涉及不那么直观的逻辑，因此需要一个专门的组件来负责对象的构建。

我猜事先值得一提的是，虽然我说构建器模式关注的是复杂对象，但我们将通过一个相当简单的例子来探讨它。这样做纯粹是为了空间优化，使得领域逻辑的复杂性不会干扰读者对模式实际实现的理解。

## Scenario

让我们想象一下，我们正在构建一个渲染网页的组件。首先，我们将输出一个包含两个项目（内容为“hello”和“world”）的简单无序列表。一个非常简化的实现可能如下所示：

```c++
string words[] = { "hello", "world" };
ostringstream oss;
oss << "<ul>";
for (auto w : words)
  oss << " <li>" << w << "</li>";
oss << "</ul>";
printf(oss.str().c_str());
```

这确实给了我们想要的结果，但这种方法不够灵活。我们如何将这个无序列表改为有序列表呢？我们如何在列表创建后添加另一个项目呢？显然，在我们这种僵化的方案中，这是不可能的。

因此，我们可能会选择面向对象编程（OOP）的方法，并定义一个 `HtmlElement` 类来存储每个标签的信息：

```c++
struct HtmlElement
{
  string name;
  string text;
  vector<HtmlElement> elements;

  HtmlElement() {}
  HtmlElement(const string& name, const string& text)
    : name(name), text(text) { }

  string str(int indent = 0) const
  {
    // pretty-print the contents
  }
};
```

借助这种方法，我们现在可以以更合理的方式创建我们的列表：

```c++
string words[] = { "hello", "world" };
HtmlElement list{"ul", ""};
for (auto w : words)
  list.elements.emplace_back{HtmlElement{"li", w}};
printf(list.str().c_str());
```

这种方法工作得很好，给我们提供了一个更可控的、面向对象驱动的项目列表表示。但是，构建每个 `HtmlElement` 的过程并不非常方便，我们可以通过实现构建器（Builder）模式来改进它。

## Simple Builder

构建器（Builder）模式仅仅试图将对象的分步构建外包给一个单独的类。我们的第一次尝试可能会得到如下结果：

```c++
struct HtmlBuilder
{
  HtmlElement root;

  HtmlElement(string root_name) { root.name = root_name; }

  void add_child(string child_name, string child_text)
  {
    HtmlElement e{ child_name, chile_text };
    root.elements.emplace_back(e);
  }

  string str() { return root.str(); }
};
```

这是一个专门用于构建 HTML 元素的组件。`add_child()` 方法是用来向当前元素添加额外子元素的方法，每个子元素是一个名称-文本对。它可以如下使用：

```c++
HtmlBuilder builder{ "ul" };
builder.add_child("li", "hello");
builder.add_child("li", "world");
cout << builder.str() << endl;
```

您会注意到，目前 `add_child()` 函数是返回 `void` 的。我们可以利用返回值做很多事情，但返回值的一个最常见用途是帮助我们构建一个流畅接口（fluent interface）。

## Fluent Builder

让我们将 `add_child()` 的定义更改为如下：

```c++
HtmlBuilder& add_child(string child_name, string child_text)
{
  HtmlElement e{ child_name, child_text };
  root.elements.emplace_back(e);
  return *this;
}
```

通过返回构建器本身的引用，现在可以将构建器调用链式连接。这被称为流畅接口（fluent interface）：

```c++
HtmlBuilder builder{ "ul" };
builder.add_child("li", "hello").add_child("li", "world");
cout << builder.str() << endl;
```

选择引用或指针完全取决于您。如果您想使用 `->` 操作符链式调用，您可以如下定义 `add_child()`：

```c++
HtmlBuilder* add_child(string child_name, string child_text)
{
  HtmlElement e{ child_name, child_text };
  root.elements.emplace_back(e);
  return this;
}
```

然后可以像这样使用它：

```c++
HtmlBuilder builder{"ul"};
builder->add_child("li", "hello")->add_child("li", "world");
cout << builder << endl;
```

## Communicating Intent

我们已经为 HTML 元素实现了一个专门的构建器，但我们的类用户将如何知道如何使用它呢？一个想法是，在他们构造对象时，简单地强制他们使用构建器。您需要做的是：

```c++
struct HtmlElement
{
  string name;
  string text;
  vector<HtmlElement> elements;
  const size_t indent_size = 2;

  static unique_ptr<HtmlBuilder> build(const string& root_name)
  {
    return make_unique<HtmlBuilder>(root_name);
  }

protected: // hide all constructors
  HtmlElement() {}
  HtmlElement(const string& name, const string& text)
    : name{name}, text{text}
  {
  }
};
```

我们的方法是双管齐下的。首先，我们隐藏了所有构造函数，因此它们不再可用。然而，我们创建了一个工厂方法（这是一种我们稍后会讨论的设计模式），可以直接从 HtmlElement 创建构建器。而且这个方法是静态的。以下是使用它的方法：

```c++
auto builder = HtmlElement::build("ul");
builder.add_child("li", "hello").add_child("li", "world");
cout << builder.str() << endl;
```

但不要忘了我们最终的目标是构建一个 `HtmlElement`，而不仅仅是一个构建器！因此，锦上添花的做法是在构建器上实现 `operator HtmlElement`，以得出最终值：

```c++
struct HtmlBuilder
{
  operator HtmlBuilder() const { return root; }
  HtmlBuilder root;
  // other operations omitted
};
```

对前述内容的一个变体是返回 `std::move(root)`，但是否这样做完全取决于您。

无论如何，添加该操作符后，我们可以编写如下代码：

```c++
HtmlElement e = HtmlElement::build("ul")
  .add_child("li", "hello")
  .add_child("li", "world");
cout << e.str() << endl;
```

遗憾的是，没有明确的方法告诉其他用户以这种方式使用 API。希望构造函数的限制以及静态 `build()` 函数的存在能够引导用户使用构建器，但除了该操作符之外，在 `HtmlBuilder` 本身中添加一个对应的 `build()` 函数也可能是一个明智的选择：

```c++
HtmlElement HtmlBuilder::build() const
{
  return root; // again, std::move possible here
}
```

## Groovy-Style Builder

这个例子稍微偏离了专门的构建器，因为实际上这里并没有显式的构建器。它只是对象构造的一种替代方法。

像 Groovy、Kotlin 等编程语言都试图通过支持使 DSL（领域特定语言）构建过程更佳的语法结构来展示它们的优势。但为什么 C++ 应该有所不同呢？借助初始化列表，我们可以有效地使用普通类构建一个 HTML-compatible DSL（领域特定语言）。

首先，我们将定义一个 HTML 标签：

```c++
struct Tag{
  std::string name;
  std::string text;
  std::vector<Tag> children;
  std::vector<std::pair<std::string, std::string>> attributes;  

  friend std::ostream& operator<<(std::ostream& os, const Tag& tag)
  {
    // implementation omitted
  }
};
```

到目前为止，我们已经有一个可以存储其名称、文本、子元素（内部标签）甚至 HTML 属性的 `Tag` 类。我们还有一些美化输出的代码，但这些代码过于枯燥，在此不予展示。

现在我们可以为它添加几个 `protected` 的构造函数（因为我们不希望任何人直接实例化这个类）。之前的实验告诉我们至少有两种情况：

- 通过名称和文本初始化的标签（例如，列表项）
- 通过名称和一系列子元素初始化的标签

第二种情况更有趣；我们将使用类型为 `std::vector` 的参数：