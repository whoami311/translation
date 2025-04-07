# Boost.Signals2

## Introduction

Boost.Signals2 库是托管信号与槽系统的实现。信号代表有多个目标的回调函数，在类似的系统中也被称为发布者或事件。信号连接到一组槽，这些槽是回调接收器（也称为事件目标或订阅者），当信号被 “发射” 时，这些槽会被调用。

信号和槽是被管理的，这意味着信号和槽（或者更恰当地说，作为槽一部分的对象）可以追踪连接，并且能够在任一相关对象被销毁时自动断开信号与槽之间的连接。这使得用户可以在不花费大量精力管理所有涉及对象生命周期的情况下，进行信号和槽的连接。

当一个信号连接到多个槽时，关于槽的返回值与信号本身的返回值之间的关系存在一个问题。Boost.Signals2 允许用户指定如何合并多个返回值的方式。

### Signals2

本文档描述了原始 Boost.Signals 库的一个线程安全变体。为了支持线程安全性，接口做了一些改动，主要涉及自动连接管理。此实现由 Frank Mori Hess 编写。同时感谢 Timmo Stange、Peter Dimov 和 Tony Van Eerd 提供的想法和反馈，也感谢 Douglas Gregor 编写的 Boost.Signals 原始版本，本工作是基于该版本进行的。

## Tutorial

### How to Read this Tutorial

本教程并非设计为线性阅读。其顶层结构大致将库中的不同概念分开（例如，处理调用多个槽、向槽传递值以及从槽返回值），在每个概念中，首先介绍基本思想，然后描述库的更复杂用法。每个部分都标记为*初级*、*中级*或*高级*，以帮助读者选择。*初级*部分包含所有库用户都应该了解的信息；仅阅读*初级*部分即可很好地使用 Signals2 库。*中级*部分基于*初级*部分，介绍库的一些稍微复杂的用法。最后，*高级*部分详细说明 Signals2 库的非常高级的用法，这些内容通常需要扎实掌握*初级*和*中级*主题的知识；大多数用户不需要阅读*高级*部分。

### Hello, World! (Beginner)

以下示例使用信号和槽来输出 "Hello, World!"。首先，我们创建一个信号 `sig`，这是一个不接受参数且返回值为 `void` 的信号。接下来，我们使用 `connect` 方法将 `hello` 函数对象连接到该信号。最后，像调用函数一样使用信号 `sig` 来调用槽，这会反过来调用 `HelloWorld::operator()` 以打印 "Hello, World!"。

```c++
struct HelloWorld
{
  void operator()() const
  {
    std::cout << "Hello, World!" << std::endl;
  }
};
```

```c++
  // Signal with no arguments and a void return value
  boost::signals2::signal<void ()> sig;

  // Connect a HelloWorld slot
  HelloWorld hello;
  sig.connect(hello);

  // Call all of the slots
  sig();
```

### Calling Multiple Slots

#### Connecting Multiple Slots (Beginner)

从信号调用单个槽并不是非常有趣，因此我们可以通过将打印 "Hello, World!" 的工作分成两个完全独立的槽，使 Hello, World 程序变得更有趣。第一个槽将打印 "Hello"，其代码可能如下所示：

```c++
struct Hello
{
  void operator()() const
  {
    std::cout << "Hello";
  }
};
```

第二个槽将打印 ", World!" 和一个换行符，以完成程序。第二个槽的代码可能如下所示：

```c++
struct World
{
  void operator()() const
  {
    std::cout << ", World!" << std::endl;
  }
};
```

像我们之前的示例一样，我们可以创建一个不接受参数且返回值为 `void` 的信号 `sig`。这次，我们将 `hello` 和 `world` 两个槽连接到同一个信号上，当我们调用该信号时，这两个槽都会被调用。

```c++
  boost::signals2::signal<void ()> sig;

  sig.connect(Hello());
  sig.connect(World());

  sig();
```

默认情况下，槽会被添加到槽列表的末尾，因此该程序的输出将符合预期：

```
Hello, World!
```

#### Ordering Slot Call Groups (Intermediate)

槽可以自由地包含副作用，这意味着即使某些槽的连接顺序不同，它们也必须在其他槽之前被调用。Boost.Signals2 库允许将槽分组，并对这些组进行排序。对于我们的 Hello, World 程序，我们希望 "Hello" 在 ", World!" 之前打印，因此我们将 "Hello" 放入一个必须在包含 ", World!" 的组之前执行的组中。为此，我们可以在 `connect` 调用的开头提供一个额外参数来指定组。默认情况下，组值是 `int` 类型，并按照整数的 `<` 关系进行排序。以下是构建 Hello, World 程序的方法：

```c++
  boost::signals2::signal<void ()> sig;

  sig.connect(1, World());  // connect with group 1
  sig.connect(0, Hello());  // connect with group 0
```

调用信号将正确打印 "Hello, World!"，因为 `Hello` 对象位于组 `0` 中，这先于 `World` 对象所在的组 `1`。组参数实际上是可选的。在第一个 Hello, World 示例中我们省略了它，因为当所有槽都是独立的时候这一步是不必要的。那么，如果我们混合使用带有组参数和不带组参数的 `connect` 调用会发生什么呢？“未命名”的槽（即，那些在连接时没有指定组名的槽）可以被放置在槽列表的前面或后面（通过分别传递 `boost::signals2::at_front` 或 `boost::signals2::at_back` 作为 `connect` 的最后一个参数），默认情况下它们会被放在列表的末尾。当指定了一个组时，最终的 `at_front` 或 `at_back` 参数描述了槽在组排序中的位置。使用 `at_front` 连接的非分组槽总是先于所有分组槽。使用 `at_back` 连接的非分组槽总是位于所有分组槽之后。

如果我们像这样向示例中添加一个新的槽：

```c++
struct GoodMorning
{
  void operator()() const
  {
    std::cout << "... and good morning!" << std::endl;
  }
};
```

```c++
  // by default slots are connected at the end of the slot list
  sig.connect(GoodMorning());

  // slots are invoked this order:
  // 1) ungrouped slots connected with boost::signals2::at_front
  // 2) grouped slots according to ordering of their groups
  // 3) ungrouped slots connected with boost::signals2::at_back
  sig();
```

... 我们将得到我们期望的结果：

```
Hello, World!
... and good morning!
```

### Passing Values to and from Slots

#### Slot Arguments (Beginner)

信号可以将参数传递给它们调用的每个槽。例如，传播鼠标移动事件的信号可能希望传递新的鼠标坐标以及鼠标按钮是否被按下。

作为示例，我们将创建一个传递两个 `float` 类型参数给其槽的信号。然后，我们将创建一些槽来打印对这些值进行各种算术运算的结果。

```c++
void print_args(float x, float y)
{
  std::cout << "The arguments are " << x << " and " << y << std::endl;
}

void print_sum(float x, float y)
{
  std::cout << "The sum is " << x + y << std::endl;
}

void print_product(float x, float y)
{
  std::cout << "The product is " << x * y << std::endl;
}

void print_difference(float x, float y)
{
  std::cout << "The difference is " << x - y << std::endl;
}

void print_quotient(float x, float y)
{
  std::cout << "The quotient is " << x / y << std::endl;
}
```

```c++
  boost::signals2::signal<void (float, float)> sig;

  sig.connect(&print_args);
  sig.connect(&print_sum);
  sig.connect(&print_product);
  sig.connect(&print_difference);
  sig.connect(&print_quotient);

  sig(5., 3.);
```

该程序将输出以下内容：

```
The arguments are 5 and 3
The sum is 8
The product is 15
The difference is 2
The quotient is 1.66667
```

因此，当像函数一样调用 `sig` 时，传递给它的任何值都会被传递到每个槽。在创建信号时，我们需要提前声明这些值的类型。类型 `boost::signals2::signal<void (float, float)>` 表示该信号具有 `void` 返回值，并接受两个 `float` 类型的参数。因此，连接到 `sig` 的任何槽都必须能够接受两个 `float` 类型的参数。

#### Signal Return Values (Advanced)

正如槽可以接收参数一样，它们也可以返回值。这些值可以通过组合器（combiner）返回给信号的调用者。组合器是一种机制，它能够获取调用槽的结果（可能没有结果，也可能有上百个结果，只有在程序运行时才能知道），并将它们合并为一个单一的结果返回给调用者。单一结果通常是槽调用结果的简单函数：例如最后一个槽调用的结果、任何槽返回的最大值，或者包含所有结果的容器等。

我们可以稍微修改之前的算术运算示例，使得槽都返回计算乘积、商、和或差的结果。然后，信号本身可以根据这些结果返回一个值，并将其打印出来：

```c++
float product(float x, float y) { return x * y; }
float quotient(float x, float y) { return x / y; }
float sum(float x, float y) { return x + y; }
float difference(float x, float y) { return x - y; }
```

```c++
boost::signals2::signal<float (float, float)> sig;
```

```c++
  sig.connect(&product);
  sig.connect(&quotient);
  sig.connect(&sum);
  sig.connect(&difference);

  // The default combiner returns a boost::optional containing the return
  // value of the last slot in the slot list, in this case the
  // difference function.
  std::cout << *sig(5, 3) << std::endl;
```

该示例程序将输出 `2`。这是因为具有返回类型（`float`，传递给 `boost::signals2::signal` 类模板的第一个模板参数）的信号默认行为是调用所有槽，并返回一个包含最后一个被调用槽的结果的 `boost::optional`。对于这个示例来说，这种行为确实有些愚蠢，因为槽没有副作用，结果就是最后连接的槽的返回值。

更有趣的信号结果是所有槽返回值的最大值。为此，我们可以创建一个自定义组合器，其代码如下：

```c++
// combiner which returns the maximum value returned by all slots
template<typename T>
struct maximum
{
  typedef T result_type;

  template<typename InputIterator>
  T operator()(InputIterator first, InputIterator last) const
  {
    // If there are no slots to call, just return the
    // default-constructed value
    if(first == last ) return T();
    T max_value = *first++;
    while (first != last) {
      if (max_value < *first)
        max_value = *first;
      ++first;
    }

    return max_value;
  }
};
```

`maximum` 类模板充当函数对象。其结果类型由其模板参数给出，这也是它期望计算最大值的类型（例如，`maximum<float>` 会找到一系列 `float` 中的最大值）。当一个 `maximum` 对象被调用时，它会被提供一个输入迭代器序列 `[first, last)`，该序列包含调用所有槽的结果。`maximum` 使用这个输入迭代器序列来计算最大元素，并返回那个最大值。

我们通过将其作为信号的组合器来实际使用这个新的函数对象类型。组合器模板参数跟随信号的调用签名之后：

```c++
boost::signals2::signal<float (float x, float y),
              maximum<float> > sig;
```

现在我们可以连接执行算术函数的槽，并使用该信号：

```c++
  sig.connect(&product);
  sig.connect(&quotient);
  sig.connect(&sum);
  sig.connect(&difference);

  // Outputs the maximum value returned by the connected slots, in this case
  // 15 from the product function.
  std::cout << "maximum: " << sig(5, 3) << std::endl;
```

该程序的输出将是 `15`，因为无论槽以何种顺序连接，`5` 和 `3` 的乘积都比商、和或差更大。

在其他情况下，我们可能希望将所有槽计算的值一起返回，放在一个大型数据结构中。这可以通过使用不同的组合器轻松实现：

```c++
// aggregate_values is a combiner which places all the values returned
// from slots into a container
template<typename Container>
struct aggregate_values
{
  typedef Container result_type;

  template<typename InputIterator>
  Container operator()(InputIterator first, InputIterator last) const
  {
    Container values;

    while(first != last) {
      values.push_back(*first);
      ++first;
    }
    return values;
  }
};
```

同样，我们可以使用这个新的组合器创建一个信号：

```c++
boost::signals2::signal<float (float, float),
    aggregate_values<std::vector<float> > > sig;
```

```c++
  sig.connect(&quotient);
  sig.connect(&product);
  sig.connect(&sum);
  sig.connect(&difference);

  std::vector<float> results = sig(5, 3);
  std::cout << "aggregate values: ";
  std::copy(results.begin(), results.end(),
    std::ostream_iterator<float>(std::cout, " "));
  std::cout << "\n";
```

该程序的输出将包含 `15`、`8`、`1.6667` 和 `2`。这里有趣的是，信号类的第一个模板参数 `float` 实际上并不是信号的返回类型。相反，它是连接的槽使用的返回类型，并且也将是传递给组合器的输入迭代器的 `value_type`。组合器本身是一个函数对象，其 `result_type` 成员类型成为信号的返回类型。

传递给组合器的输入迭代器将解引用操作转换为槽调用。因此，组合器可以选择仅调用某些槽直到满足特定条件为止。例如，在分布式计算系统中，组合器可以询问每个远程系统是否它将处理请求。只需要一个远程系统处理特定请求，所以在某个远程系统接受了任务之后，我们不希望再让其他远程系统执行相同任务。这样的组合器只需检查解引用迭代器时返回的值，并在值可接受时返回。下面的组合器返回指向 `FulfilledRequest` 数据结构的第一个非空指针，而无需请求任何后续槽来完成请求：

```c++
struct DistributeRequest {
  typedef FulfilledRequest* result_type;

  template<typename InputIterator>
  result_type operator()(InputIterator first, InputIterator last) const
  {
    while (first != last) {
      if (result_type fulfilled = *first)
        return fulfilled;
      ++first;
    }
    return 0;
  }
};
```

### Connection Management

#### Disconnecting Slots (Beginner)

槽在连接后并不期望无限期存在。通常，槽仅用于接收少量事件后就会被断开连接，程序员需要控制决定何时不再连接某个槽。

管理连接的入口是 `boost::signals2::connection` 类。`connection` 类唯一地表示特定信号与特定槽之间的连接。`connected()` 方法检查信号和槽是否仍然连接，而 `disconnect()` 方法在调用前如果信号和槽还处于连接状态，则会断开它们的连接。每次调用信号的 `connect()` 方法都会返回一个 `connection` 对象，该对象可用于确定连接是否仍然存在或断开信号和槽的连接。

```c++
  boost::signals2::connection c = sig.connect(HelloWorld());
  std::cout << "c is connected\n";
  sig(); // Prints "Hello, World!"

  c.disconnect(); // Disconnect the HelloWorld object
  std::cout << "c is disconnected\n";
  sig(); // Does nothing: there are no connected slots
```

#### Blocking Slots (Beginner)

槽可以被临时“阻塞”，这意味着当信号被调用时它们将被忽略，但并未被永久断开连接。这通常用于防止无限递归的情况，例如运行某个槽可能会导致它所连接的信号再次被触发。`boost::signals2::shared_connection_block` 对象可以临时阻塞一个槽。通过销毁或对所有引用该连接的 `shared_connection_block` 对象调用 `unblock` 方法，可以解除连接的阻塞。以下是一个阻塞/解除阻塞槽的示例：

```c++
  boost::signals2::connection c = sig.connect(HelloWorld());
  std::cout << "c is not blocked.\n";
  sig(); // Prints "Hello, World!"

  {
    boost::signals2::shared_connection_block block(c); // block the slot
    std::cout << "c is blocked.\n";
    sig(); // No output: the slot is blocked
  } // shared_connection_block going out of scope unblocks the slot
  std::cout << "c is not blocked.\n";
  sig(); // Prints "Hello, World!"}
```

#### Scoped Connections (Intermediate)

`boost::signals2::scoped_connection` 类引用一个信号/槽连接，当 `scoped_connection` 类超出作用域时，该连接将被断开。这种能力在连接仅需临时存在时非常有用，例如：

```c++
  {
    boost::signals2::scoped_connection c(sig.connect(ShortLived()));
    sig(); // will call ShortLived function object
  } // scoped_connection goes out of scope and disconnects

  sig(); // ShortLived function object no longer connected to sig
```

注意，尝试使用赋值语法初始化 `scoped_connection` 将会失败，因为它不可复制。可以使用显式初始化语法或者先默认构造再从 `signals2::connection` 进行赋值的方式来实现：

```c++
// doesn't compile due to compiler attempting to copy a temporary scoped_connection object
// boost::signals2::scoped_connection c0 = sig.connect(ShortLived());

// okay
boost::signals2::scoped_connection c1(sig.connect(ShortLived()));

// also okay
boost::signals2::scoped_connection c2;
c2 = sig.connect(ShortLived());
```

#### Disconnecting Equivalent Slots (Intermediate)

可以使用 `signal::disconnect` 方法的一种形式来断开与给定函数对象等价的槽，但前提是该函数对象的类型具有可访问的 `==` 运算符。例如：

```c++
void foo() { std::cout << "foo"; }
void bar() { std::cout << "bar\n"; }
```

```c++
boost::signals2::signal<void ()> sig;
```

```c++
  sig.connect(&foo);
  sig.connect(&bar);
  sig();

  // disconnects foo, but not bar
  sig.disconnect(&foo);
  sig();
```

#### Automatic Connection Management (Intermediate)

Boost.Signals2 能够自动跟踪涉及信号/槽连接的对象的生命周期，包括在涉及槽调用的对象被销毁时自动断开槽的连接。例如，考虑一个简单的新闻传递服务，客户端连接到新闻提供者，当有信息到达时，新闻提供者会将新闻发送给所有已连接的客户端。新闻传递服务可以这样构建：

```c++
class NewsItem { /* ... */ };

typedef boost::signals2::signal<void (const NewsItem&)> signal_type;
signal_type deliverNews;
```

希望接收新闻更新的客户端只需将一个可以接收新闻项的函数对象连接到 `deliverNews` 信号即可。例如，我们的应用程序中可能有一个专门用于新闻的特殊消息区域，例如：

```c++
struct NewsMessageArea : public MessageArea
{
public:
  // ...

  void displayNews(const NewsItem& news) const
  {
    messageText = news.text();
    update();
  }
};

// ...
NewsMessageArea *newsMessageArea = new NewsMessageArea(/* ... */);
// ...
deliverNews.connect(boost::bind(&NewsMessageArea::displayNews,
  newsMessageArea, _1));
```

然而，如果用户关闭了新闻消息区域，导致 `deliverNews` 所知的 `newsMessageArea` 对象被销毁，会发生什么？最有可能的结果是发生段错误。然而，通过 Boost.Signals2，可以使用 `slot::track` 来跟踪任何由 `shared_ptr` 管理的对象。当槽所跟踪的任何对象过期时，该槽会自动断开连接。此外，Boost.Signals2 会确保在与其关联的槽执行过程中，不会有任何被跟踪的对象过期。它通过在执行槽之前创建槽所跟踪对象的临时 `shared_ptr` 副本来实现这一点。为了跟踪 `NewsMessageArea`，我们使用 `shared_ptr` 来管理其生命周期，并在连接之前通过槽的 `slot::track` 方法将 `shared_ptr` 传递给槽，例如：

```c++
// ...
boost::shared_ptr<NewsMessageArea> newsMessageArea(new NewsMessageArea(/* ... */));
// ...
deliverNews.connect(signal_type::slot_type(&NewsMessageArea::displayNews,
  newsMessageArea.get(), _1).track(newsMessageArea));
```

注意，在上面的例子中不需要显式调用 `bind()`。如果 `signals2::slot` 构造函数被传递了多于一个参数，它会自动将所有参数传递给 `bind` 并使用返回的函数对象。

另外请注意，我们在槽构造函数中作为第二个参数传递的是一个普通指针，使用 `newsMessageArea.get()` 而不是直接传递 `shared_ptr` 本身。如果我们传递了 `newsMessageArea` 自身，那么 `shared_ptr` 的副本将会绑定到槽函数中，阻止 `shared_ptr` 过期。然而，使用 `slot::track` 暗示我们希望允许跟踪的对象过期，并在发生这种情况时自动断开连接。

除了 `boost::shared_ptr` 之外的 `shared_ptr` 类（如 `std::shared_ptr`）也可以为了连接管理的目的进行跟踪。它们可以通过 `slot::track_foreign` 方法来支持。

#### Postconstructors and Predestructors (Advanced)

使用 `shared_ptr` 进行跟踪的一个限制是，对象不能在其构造函数中设置自身的跟踪。然而，可以在一个后构造函数（post-constructor）中设置跟踪，该后构造函数在对象创建并传递给 `shared_ptr` 后被调用。Boost.Signals2 库通过 `deconstruct()` 工厂函数提供对后构造函数和前析构函数的支持。

对于大多数情况，为类设置后构造函数最简单且最稳健的方法是定义一个关联的 `adl_postconstruct` 函数，该函数可以被 `deconstruct()` 查找，将类的构造函数设为私有，并通过声明 `deconstruct_access` 作为友元来给予 `deconstruct` 访问私有构造函数的权限。这将确保只能通过 `deconstruct()` 函数创建该类的对象，并且其关联的 `adl_postconstruct()` 函数总会被调用。

示例部分包含多个定义带有后构造函数和前析构函数的类的例子，以及如何使用 `deconstruct()` 创建这些类的对象。

需要注意的是，Boost.Signals2 中的后构造函数/前析构函数支持并不是使用该库所必需的。使用 `deconstruct` 是完全可选的。一种替代方案是为你的类定义静态工厂函数。工厂函数可以创建一个对象，将对象的所有权传递给 `shared_ptr`，为对象设置跟踪，然后返回 `shared_ptr`。

#### When Can Disconnections Occur? (Intermediate)

信号/槽断开连接会在以下任一条件发生时触发：

- 连接通过连接的 `disconnect` 方法直接显式断开，或通过信号的 `disconnect` 方法间接断开，或通过 `scoped_connection` 的析构函数断开。
- 被槽跟踪的对象被销毁。
- 信号被销毁。

这些事件可以在任何时候发生，而不会中断信号的调用序列。如果在信号的调用序列期间某个信号/槽连接被断开，调用序列仍将继续，但不会调用已断开的槽。此外，信号可能在其调用序列中被销毁，在这种情况下，它将完成其槽调用序列，但无法再被直接访问。

信号可以递归调用（例如，信号 A 调用槽 B，而槽 B 又调用了信号 A...）。在递归情况下，断开行为不会改变，只是槽调用序列包括该信号所有嵌套调用的槽调用。

需要注意的是，即使连接已断开，其关联的槽可能仍在执行过程中。换句话说，断开连接不会阻塞等待关联槽完成执行。这种情况可能在多线程环境中发生，如果断开操作与信号调用并发进行；或者在单线程环境中发生，如果槽断开了自身。

#### Passing Slots (Intermediate)

在 Boost.Signals2 库中，槽是从任意函数对象创建的，因此没有固定的类型。然而，通常需要将槽通过不能是模板的接口传递。可以通过每个特定信号类型的 `slot_type` 来传递槽，并且任何与信号签名兼容的函数对象都可以传递给 `slot_type` 参数。例如：

```c++
// a pretend GUI button
class Button
{
  typedef boost::signals2::signal<void (int x, int y)> OnClick;
public:
  typedef OnClick::slot_type OnClickSlotType;
  // forward slots through Button interface to its private signal
  boost::signals2::connection doOnClick(const OnClickSlotType & slot);

  // simulate user clicking on GUI button at coordinates 52, 38
  void simulateClick();
private:
  OnClick onClick;
};

boost::signals2::connection Button::doOnClick(const OnClickSlotType & slot)
{
  return onClick.connect(slot);
}

void Button::simulateClick()
{
  onClick(52, 38);
}

void printCoordinates(long x, long y)
{
  std::cout << "(" << x << ", " << y << ")\n";
}
```

```c++
  Button button;
  button.doOnClick(&printCoordinates);
  button.simulateClick();
```

`doOnClick` 方法现在在功能上等同于 `onClick` 信号的 `connect` 方法，但是 `doOnClick` 方法的细节可以隐藏在实现细节文件中。

### Example: Document-View

信号可以用于实现灵活的文档-视图架构。文档将包含一个信号，每个视图都可以连接到这个信号。下面的 `Document` 类定义了一个简单的文本文档，支持多个视图。请注意，它存储了一个所有视图都将连接的信号。

```c++
class Document
{
public:
    typedef boost::signals2::signal<void ()>  signal_t;

public:
    Document()
    {}

    /* Connect a slot to the signal which will be emitted whenever
      text is appended to the document. */
    boost::signals2::connection connect(const signal_t::slot_type &subscriber)
    {
        return m_sig.connect(subscriber);
    }

    void append(const char* s)
    {
        m_text += s;
        m_sig();
    }

    const std::string& getText() const
    {
        return m_text;
    }

private:
    signal_t    m_sig;
    std::string m_text;
};
```

接下来，我们可以开始定义视图。以下 `TextView` 类提供了文档文本的简单视图。

```c++
class TextView
{
public:
    TextView(Document& doc): m_document(doc)
    {
        m_connection = m_document.connect(boost::bind(&TextView::refresh, this));
    }

    ~TextView()
    {
        m_connection.disconnect();
    }

    void refresh() const
    {
        std::cout << "TextView: " << m_document.getText() << std::endl;
    }
private:
    Document&               m_document;
    boost::signals2::connection  m_connection;
};
```

或者，我们可以使用 `HexView` 视图提供文档的十六进制值转换视图：

```c++
class HexView
{
public:
    HexView(Document& doc): m_document(doc)
    {
        m_connection = m_document.connect(boost::bind(&HexView::refresh, this));
    }

    ~HexView()
    {
        m_connection.disconnect();
    }

    void refresh() const
    {
        const std::string&  s = m_document.getText();

        std::cout << "HexView:";

        for (std::string::const_iterator it = s.begin(); it != s.end(); ++it)
            std::cout << ' ' << std::hex << static_cast<int>(*it);

        std::cout << std::endl;
    }
private:
    Document&               m_document;
    boost::signals2::connection  m_connection;
};
```

为了将这个示例整合起来，下面是一个简单的 `main` 函数，它设置了两个视图，然后对文档进行了修改：

```c++
int main(int argc, char* argv[])
{
    Document    doc;
    TextView    v1(doc);
    HexView     v2(doc);

    doc.append(argc == 2 ? argv[1] : "Hello world!");
    return 0;
}
```

完整的示例源代码，由 Keith MacDonald 提供，可以在示例部分找到。我们还提供了该程序的几种变体，它们使用自动连接管理在视图销毁时自动断开连接。

### Giving a Slot Access to its Connection (Advanced)

您可能会遇到希望从槽内部断开或阻塞槽连接的情况。例如，假设您有一组异步任务，每个任务完成时都会发出一个信号。您希望将一个槽连接到所有任务上，以便在每个任务完成时检索它们的结果。一旦某个任务完成并且槽被运行，该槽就不再需要连接到已完成的任务。因此，您可能希望通过让槽在其运行时断开其调用连接来清理旧的连接。

要让槽断开（或阻塞）其调用连接，它必须能够访问一个 `signals2::connection` 对象，该对象引用了调用信号-槽连接。难点在于，连接对象是由 `signal::connect` 方法返回的，因此直到槽已经连接到信号之后才可用。这在多线程环境中尤其麻烦，因为当槽正在连接时，信号可能被另一个线程并发调用。

因此，信号类提供了 `signal::connect_extended` 方法，允许将带有额外参数的槽连接到信号。额外的参数是一个 `signals2::connection` 对象，它引用了当前调用槽的信号-槽连接。`signal::connect_extended` 使用由 `signal::extended_slot_type` 类型定义指定类型的槽。

示例部分包含一个 `extended_slot` 程序，演示了使用 `signal::connect_extended` 的语法。

### Changing the Mutex Type of a Signal (Advanced).

在大多数情况下，`boost::signals2::mutex` 作为 `signals2::signal` 的 Mutex 模板类型参数的默认类型应该是合适的。如果您希望使用其他互斥锁类型，它必须是可默认构造的，并且符合 Boost.Thread 库定义的 Lockable 概念。也就是说，它必须有 `lock()` 和 `unlock()` 方法（Lockable 概念还包括一个 `try_lock()` 方法，但此库不需要尝试锁定）。

Boost.Signals2 库为信号提供了一个替代的互斥锁类：`boost::signals2::dummy_mutex`。这是一个用于单线程程序的假互斥锁，在这种程序中，锁定真实的互斥锁将是无用的开销。您可以与信号一起使用的其他互斥锁类型包括 `boost::mutex` 或 C++11 中的 `std::mutex`。

更改信号的 Mutex 模板类型参数可能会很繁琐，因为在其之前有大量的模板参数。在这种情况下，`signal_type` 元函数特别有用，因为它为 `signals2::signal` 类启用了命名模板类型参数。例如，要声明一个接受 `int` 作为参数并使用 `boost::signals2::dummy_mutex` 作为其 Mutex 类型的信号，您可以这样写：

```c++
namespace bs2 = boost::signals2;
using namespace bs2::keywords;
bs2::signal_type<void (int), mutex_type<bs2::dummy_mutex> >::type sig;
```

### Linking against the Signals2 library

与最初的 Boost.Signals 库不同，Boost.Signals2 目前是仅头文件的库。
