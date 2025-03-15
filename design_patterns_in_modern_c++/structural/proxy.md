# Proxy

当我们研究装饰器（Decorator）设计模式时，我们看到了增强对象功能的不同方法。代理（Proxy）设计模式与此类似，但其目标通常是尽可能精确地保留被使用的 API，同时提供某些内部增强。

代理并不是一个同质化的 API，因为人们构建的不同类型的代理相当多，并且服务于完全不同的目的。在这一章中，我们将查看几种不同的代理对象，您也可以在线找到更多相关信息。

## 智能指针

代理（Proxy）模式最简单和最直接的示例是智能指针。智能指针是指针的包装器，它同时还维护一个引用计数，重载某些运算符，但总体来说，它提供了与普通指针相同的接口：

```c++
struct BankAccount
{
  void deposit(int amount) { ... }
};

BankAccount *ba = new BankAccount;
ba->deposit(123);
auto ba2 = make_shared<BankAccount>();
ba2->deposit(123); // same API!
```

因此，智能指针也可以用作某些期望普通指针位置的替代。例如，`if (ba) { ... }` 无论是 `ba` 是普通指针还是智能指针都是有效的，在这两种情况下，`*ba` 都会获取到底层对象。依此类推。

当然，两者之间存在差异。最明显的一点是，您不需要对智能指针调用 `delete`。除此之外，它确实尽量做到与普通指针尽可能接近。

## 属性代理

在其他编程语言中，“属性”（property）一词用于表示一个字段及其对应的 `getter` 和 `setter` 方法。C++ 中并没有属性，但如果我们要继续使用字段的同时赋予它特定的访问器（accessor）/修改器（mutator）行为，我们可以构建一个属性代理（property proxy）。

本质上，属性代理是一个可以伪装成属性的类，因此我们可以像这样定义它：

```c++
template <typename T>
struct Property
{
  T value;
  Property(const T initial_value)
  {
    *this = initial_value;
  }

  operator T()
  {
    // perform some getter action
    return value;
  }

  T operator =(T new_value)
  {
    // perform some setter action
    return value = new_value;
  }
};
```

在前述的实现中，我在通常会自定义（或直接替换）的地方添加了注释，这些地方大致对应于如果您选择使用 `getter` / `setter` 方法时它们的位置。

因此，我们的 `Property<T>` 类本质上是 `T` 的一个即插即用替代品，无论 `T` 实际上是什么。它通过简单地允许 `T` 类型之间的转换并使用 `value` 字段来工作。现在您可以使用它，例如，作为字段：

```c++
struct Creature
{
  Property<int> strength{ 10 };
  Property<int> agility{ 5 };
};
```

并且对字段的典型操作同样适用于属性代理类型的字段：

```c++
Creature creature;
creature.agility = 20;
auto x = creature.strength;
```

## 虚拟代理

如果您尝试解引用一个 `nullptr` 或未初始化的指针，那您就会遇到麻烦。然而，存在一些情况下您只想在对象被访问时才构造它，并且不希望过早地分配它。

这种做法被称为懒加载（lazy instantiation）。如果您确切知道哪里需要懒加载行为，您可以提前规划并为此做出特别安排。如果不知道，那么您可以构建一个代理，该代理接收一个已存在的对象并使其具有懒加载特性。我们称这种代理为虚拟代理（virtual proxy），因为底层对象可能甚至不存在，所以与其访问具体对象，我们实际上是访问一个虚拟的对象。

想象一个典型的 Image 接口：

```c++
struct Image
{
  virtual void draw() = 0;
};
```

一个急切的（与懒加载相反）`Bitmap` 实现会在构造时从文件加载图像，即使该图像实际上并不需要用于任何地方。（是的，下面的代码是一个模拟。）

```c++
struct Bitmap : Image
{
  Bitmap(const string& filename)
  {
    cout << "Loading image from " << filename << endl;
  }

  void draw() override
  {
    cout << "Drawing image " << filename << endl;
  }
};
```

这个 `Bitmap` 的构造本身就会触发图像的加载：

```c++
Bitmap img{ "pokemon.png" }; // Loading image from pokemon.png
```

这并不是我们想要的行为。我们希望的是一种位图，它只在使用 `draw()` 方法时才加载自己。现在，我假设我们可以回到 `Bitmap` 类并使其懒加载，但假设它是固定的且不可修改（或者不可继承）。

因此，我们可以构建一个虚拟代理（virtual proxy），它将聚合原始的 `Bitmap`，提供相同的接口，并且重用原始 `Bitmap` 的功能：

```c++
struct LazyBitmap : Image
{
  LazyBitmap(const string& filename)
    : filename(filename) {}

  ~LazyBitmap() { delete bmp; }

  void draw() override
  {
    if (!bmp)
      bmp = new Bitmap(filename);
    bmp->draw();
  }

private:
  Bitmap* bmp{nullptr};
  string filename;
};
```

我们做到了。如您所见，这个 `LazyBitmap` 的构造函数要“轻量”得多：它所做的只是存储用于加载图像的文件名，仅此而已——图像实际上并不会在此时加载。

所有的魔法都发生在 `draw()` 方法中：在这里我们检查 `bmp` 指针以确定底层的（急切的！）位图是否已经被构造。如果还没有，我们就构造它，然后调用它的 `draw()` 方法来实际绘制图像。

现在想象一下，您有一个使用 `Image` 类型的 API：

```c++
void draw_image(Image& img)
{
  cout << "About to draw the image" << endl;
  img.draw();
  cout << "Done drawing the image" << endl;
}
```

我们可以使用 `LazyBitmap` 的实例代替 `Bitmap`（好呀，多态性！）来渲染图像，以懒加载的方式加载它：

```c++
LazyBitmap img{ "pokemon.png" };
draw_image(img); // image loaded here

// About to draw the image
// Loading image from pokemon.png
// Drawing image pokemon.png
// Done drawing the image
```

## 通信代理

假设您在一个类型为 `Bar` 的对象上调用成员函数 `foo()`。通常情况下，您会认为 `Bar` 是在同一台机器上分配的，并且您同样期望 `Bar::foo()` 在同一进程中执行。

现在想象一下，您做出一个设计决策，将 `Bar` 及其所有成员移动到网络上的另一台机器上。但您仍然希望旧代码能够正常工作！如果您想继续像以前一样运行，您需要一个通信代理——一个可以代理调用“over the wire（通过网络线）”并根据需要收集结果的组件。

让我们实现一个简单的乒乓服务来说明这一点。首先，我们定义一个接口：

```c++
struct Pingable
{
  virtual wstring ping(const wstring& message) = 0;
}
```

如果我们是在同一个进程中构建乒乓服务，我们可以如下实现 `Pong`：

```c++
struct Pong : Pingable
{
  wstring ping(const wstring& message) override
  {
    return message + L" pong";
  }
};
```

基本上，您向 `Pong` 发送一个请求（ping），它会在消息末尾添加单词 "pong" 并返回该消息。请注意，这里我没有使用 `ostringstream&`，而是每次调用时创建一个新的字符串：这种 API 非常简单，可以很容易地复制为一个 Web 服务。

我们现在可以尝试这个设置并看看它在进程内是如何工作的：

```c++
void tryit(Pingable& pp)
{
  wcout << pp.ping(L"ping") << "\n";
}

Pong pp;
for (int i = 0; i < 3; ++i)
{
  tryit(pp);
}
```

最终结果是我们打印 "ping pong" 三次，正如我们所期望的那样。

现在，假设您决定将 `Pingable` 服务迁移到一个遥远的 Web 服务器上。也许您甚至决定使用其他平台，比如 ASP.NET，而不是 C++：

```c++
[Route("api/[controller]")]
public class PingPongController : Controller
{
  [HttpGet("{msg}")]
  public string Get(string msg)
  {
    return msg + " pong";
  }
} // achievement unlocked: use C# in a C++ book
```

通过这种设置，我们将构建一个名为 `RemotePong` 的通信代理来替代 `Pong`。在这里，微软的 REST SDK 非常有用。

```c++
struct RemotePong : Pingable
{
  wstring ping(const wstring& message) override
  {
    wstring result;
    http_client client(U("http://localhost:9149/"));
    uri_builder builder(U("/api/pingpong/"));
    builder.append(message);
    pplx::task<wstring> task = client.request(
      methods::GET, builder.to_string())
      .then([=](http_response r)
      {
        return r.extract_string();
      });
    task.wait();
    return task.get();
  }
};
```

如果您不习惯使用 REST SDK，前述内容可能看起来有些令人困惑；除了 REST 支持外，该 SDK 还使用了 Concurrency Runtime，这是微软的一个用于并发支持的库。

通过实现这一点，我们现在可以做一个简单的更改：

```c++
RemotePong pp; // was Pong
for (int i = 0; i < 3; ++i)
{
  tryit(pp);
}
```

就是这样，您得到相同的输出，但实际的实现可能正在世界另一端的某个地方，在 Docker 容器中的 Kestrel 上运行。

## 总结

本章介绍了多种代理。与装饰器（Decorator）模式不同，代理模式并不试图通过添加新成员来扩展对象的功能（除非不得已）。它所做的只是增强现有成员的底层行为。

存在许多不同类型的代理：

- 属性代理（Property proxies）是替代字段的对象，在赋值和/或访问期间可以执行额外的操作。
- 虚拟代理（Virtual proxies）提供对底层对象的虚拟访问，可以实现诸如懒加载对象的行为。您可能会觉得正在操作一个真实对象，但底层实现可能尚未创建，并且可以根据需要进行加载。
- 通信代理（Communication proxies）允许我们改变对象的物理位置（例如，将其移到云端），但仍然可以使用几乎相同的 API。当然，在这种情况下，API 只是一个远程服务（如 REST API）的外壳。
- 日志代理（Logging proxies）允许在调用底层函数的同时执行日志记录。

还有很多其他类型的代理，而且很有可能您自己构建的代理不会落入预先存在的类别中，而是执行特定于您所在领域的一些动作。
