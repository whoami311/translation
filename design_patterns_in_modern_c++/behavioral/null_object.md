# Null Object

我们并不总是可以选择我们所使用的接口。例如，我更希望我的汽车能自己把我送到目的地，而不需要我把 100% 的注意力放在路上和旁边那些危险的疯狂驾驶者身上。软件也是如此：有时候你并不真正想要某项功能，但它已经被内置到接口中了。那么，你会怎么做？你会创建一个空对象（Null Object）。

## Scenario

假设你继承了一个使用以下接口的库：

```c++
struct Logger
{
  virtual ~Logger() = default;
  virtual void info(const string& s) = 0;
  virtual void warn(const string& s) = 0;
};
```

该库使用此接口来操作银行账户，例如：

```c++
struct BankAccount
{
  std::shared_ptr<Logger> log;
  string name;
  int balance;

  BankAccount(const std::shared_ptr<Logger>& logger,
    const string& name, int balance)
    : log{logger}, name{name}, balance{balance} { }

  // more members here
};
```

事实上，`BankAccount` 可以具有类似于以下的成员函数：

```c++
void BankAccount::deposit(int amount)
{
  balance += amount;
  log->info("Deposited $" + lexical_cast<string>(amount)
     + " to " + name + ", balance is now $" + lexical_cast<string>(balance));
}
```

那么，这里的问题是什么呢？好吧，如果你确实需要日志记录，那就没有问题了，你只需实现自己的日志记录类……


```c++
struct ConsoleLogger : Logger
{
  void info(const string& s) override
  {
    cout << "INFO: " << s << endl;
  }

  void warn(const string& s) override
  {
    cout << "WARNING!!! " << s << endl;
  }
};
```

你可以立即使用它。但如果你根本不想使用日志记录呢？

## Null Object

再次看一下 `BankAccount` 的构造函数：

```c++
BankAccount(const shared_ptr<Logger>& logger,
  const string& name, int balance)
```

由于构造函数接受一个日志记录器（logger），因此不能假设只传递一个未初始化的 `shared_ptr<Logger>()` 就可以了。`BankAccount` 可能在内部检查指针是否有效，然后再根据它进行分发，但你并不知道它确实会这样做，并且在没有额外文档的情况下无法确定。

因此，传入 `BankAccount` 的唯一合理选择是一个空对象（null object）——一个符合接口但不包含任何功能的类：

```c++
struct NullLogger : Logger
{
  void info(const string& s) override {}
  void warn(const string& s) override {}
};
```

## shared_ptr is not a Null Object

需要注意的是，`shared_ptr` 和其他智能指针类并不是空对象（null objects）。空对象是指那些能够保持正确操作（执行无操作，即 no-op）的对象。然而，对未初始化的智能指针的调用会导致程序崩溃：

```c++
shared_ptr<int> n;
int x = *n + 1; // yikes!
```

有趣的是，从调用的角度来看，没有办法使智能指针“安全”。换句话说，你不能编写一个智能指针，使得当 `foo` 未初始化时，`foo->bar()` 会神奇地变成无操作（no-op）。原因在于，无论是前缀 `*` 还是后缀 `->` 操作符，它们只是代理底层的（原始）指针。而你无法对一个指针执行无操作。

## Design Improvements

停下来思考一下：如果 `BankAccount` 在你的控制之下，你能改进接口以使其更易于使用吗？以下是一些想法：

- 在各处加入指针检查。这可以在 `BankAccount` 端解决正确性问题，但并不能阻止库用户产生困惑。记住，你仍然没有传达指针可以为 null 的信息。

- 添加默认参数值，比如 `const shared_ptr<Logger>& logger = no_logging`，其中 `no_logging` 可能是 `BankAccount` 类的一个成员。即使这样，你仍然需要在每个想要使用该对象的地方检查指针的值。

- 使用可选类型（optional type）。这种方式在惯用法上是正确的，并且能够传达意图，但它导致了传递 `optional<shared_ptr<T>>` 的复杂性以及随后检查 `optional` 是否为空的问题。

##  Implicit Null Object

还有一个激进的想法，涉及到围绕 Logger 进行两次间接调用。这个想法包括将日志记录的过程细分为调用（我们希望有一个良好的 Logger 接口）和操作（Logger 实际上执行的内容）。因此，请考虑以下方案：

```c++
struct OptionalLogger : Logger {
  shared_ptr<Logger> impl;
  static shared_ptr<Logger> no_logging;
  Logger(const shared_ptr<Logger>& logger) : impl{logger} {}
  virtual void info(const string& s) override {
    if (impl) impl->info(s); // null check here
  }
  // and similar checks for other members
};

// a static instance of a null object
shared_ptr<Logger> BankAccount::no_logging{};
```

因此，我们现在将调用与实现抽象分离开来。接下来我们要做的是重新定义 `BankAccount` 的构造函数如下：

```c++
shared_ptr<OptionalLogger> logger;
BankAccount(const string& name, int balance,
  const shared_ptr<Logger>& logger = no_logging)
  : log{make_shared<OptionalLogger>(logger)},
    name{name},
    balance{balance} { }
```

如你所见，这里有一个巧妙的策略：我们接受一个 `Logger`，但存储的是一个 `OptionalLogger`（这是代理设计模式的应用）。然后，所有对这个可选日志记录器的调用都是安全的——它们只有在底层对象可用时才会‘发生’：

```c++
BankAccount account{ "primary account", 1000 };
account.deposit(2000); // no crash
```

在前面的例子中实现的代理对象本质上是 Pimpl 惯用法的一个定制版本。

## Summary

空对象（Null Object）模式提出了一个 API 设计的问题：我们能对所依赖的对象做出哪些假设？如果我们接受一个指针（无论是原始指针还是智能指针），我们是否就有义务在每次使用时检查这个指针？

如果你觉得没有这样的义务，那么客户端实现空对象的唯一方法就是构造一个无操作（no-op）的所需接口实现，并传递那个实例。也就是说，这只有在处理函数时才有效：如果对象的字段也被使用，例如，那么你就会遇到真正的麻烦。

如果你想积极支持将空对象作为参数传递，你需要明确这一点：要么将参数类型指定为 `std::optional`，要么给参数一个暗示内置空对象的默认值（例如 `= no_logging`），或者编写文档来解释在这个位置期望什么样的值。
