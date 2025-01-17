# Singleton

单例模式是在（相对有限的）设计模式历史中最受诟病的设计模式。然而，这并不意味着您不应该使用单例：马桶刷也不是最令人愉快的工具，但有时它确实是必要的。

单例设计模式源自一个非常简单的理念，即在您的应用程序中某个组件应该只有一个实例。例如，一个将数据库加载到内存并提供只读接口的组件是单例模式的理想候选者，因为浪费内存存储多个相同的数据集是没有意义的。实际上，您的应用程序可能有约束条件，使得两个或更多个数据库实例根本无法适应内存，或者会导致内存如此匮乏以至于程序无法正常运行。

## Singleton as Global Object

对此问题的天真做法是简单地约定我们永远不会实例化这个对象，例如：

```c++
struct Database
{
  /**
  * \brief Please do not create more than one instance.
  */
  Database() {}
};
```

这种做法的问题在于，除了您的开发同事可能会简单地忽略这个约定之外，对象可以通过一些隐蔽的方式创建，使得构造函数的调用并不明显。这可以是任何方式——拷贝构造函数 / 拷贝赋值运算符、`make_unique()` 调用，或者是使用控制反转（IoC）容器。

最显而易见的想法是提供一个单一的静态全局对象：

```c++
static Database database{};
```

全局静态对象的问题在于它们在不同编译单元中的初始化顺序是未定义的。这可能导致一些棘手的问题，比如一个全局对象引用了另一个尚未初始化的全局对象。还有一个问题是可发现性：客户端如何知道某个全局变量的存在？发现类相对容易一些，因为“转到类型”（**Go to Type**）提供的选项集比 `::` 后的自动完成功能要小得多。

缓解这一问题的一种方法是提供一个全局（或成员）函数来暴露必要的对象：

```c++
Database& get_database()
{
  static Database database;
  return database;
}
```

这个函数可以被调用来获取对数据库的引用。然而，您应该注意到，上述方法的线程安全性仅从 C++11 开始得到保证，并且您应该检查您的编译器是否确实准备插入锁以防止在静态对象初始化期间的并发访问。

当然，这种场景很容易出现问题：如果 `Database` 在其析构函数中决定使用其他同样暴露的单例，程序很可能会崩溃。这引发了一个更哲学性的问题：单例是否可以引用其他单例？

## Classic Implementation

前面的实现完全忽视了一个方面，即防止构造额外的对象。拥有一个全局静态的 `Database` 并不能真正阻止任何人创建另一个实例。

我们可以很容易地让那些想创建多于一个对象实例的人感到为难：只需在构造函数中放置一个静态计数器，并在计数值增加时抛出异常：

```c++
struct Database
{
  Database()
  {
    static int instance_count{ 0 };
    if (++instance_count > 1)
      throw std::exception("Cannot make >1 database!");
  }
};
```

这是一种特别敌对的处理方式：尽管通过抛出异常防止了创建多于一个实例，但它未能传达我们不希望任何人多次调用构造函数的事实。

防止显式构造 `Database` 的唯一方法是再次将其构造函数设为 `private`，并引入前述函数作为成员函数来返回唯一的一个实例：

```c++
struct Database
{
protected:
  Database() { /* do what you need to do */ }
public:
  static Database& get()
  {
    // thread-safe in C++11
    static Database database;
    return database;
  }
  Database(Database const&) = delete;
  Database(Database&&) = delete;
  Database& operator=(Database const&) = delete;
  Database& operator=(Database &&) = delete;
};
```

请注意，我们通过隐藏构造函数并删除拷贝 / 移动构造函数和赋值运算符，完全移除了创建 `Database` 实例的任何可能性。

在 C++11 之前，您只需将拷贝构造函数 / 拷贝赋值运算符设为 `private` 以达到大致相同的目的。作为手动实现的替代方案，您可以考虑使用 `boost::noncopyable`，这是一个可以继承的类，它添加了大致相同的成员隐藏定义……只是它不影响移动构造函数 / 拷贝赋值运算符。

我再次重申，如果数据库依赖于其他静态或全局变量，在其析构函数中使用它们是不安全的，因为这些对象的销毁顺序不是确定的，您可能会调用已经被销毁的对象。

最后，作为一个特别棘手的技巧，您可以将 `get()` 实现为堆分配（这样只有指针而不是整个对象是静态的）。

```c++
static Database& get() {
  static Database* database = new Database();
  return *database;
}
```

前述实现依赖于 `Database` 会一直存活到程序结束的假设，使用指针而非引用确保了即使您实现了析构函数（如果实现了，则必须是 `public`），该析构函数也永远不会被调用。而且，前述代码并不会导致内存泄漏。

### Thread Safety

正如我之前提到的，自 C++11 以来，以先前列出的方式初始化单例是线程安全的，这意味着如果两个线程同时调用 `get()`，我们不会遇到数据库被创建两次的情况。

在 C++11 之前，您会使用一种称为双重检查锁定（double-checked locking）的方法来构造单例。一个典型的实现看起来像这样：

```c++
struct Database
{
  // same members as before, but then...
  static Database& instance();
private:
  static boost::atomic<Database*> instance;
  static boost::mutex mtx;
};

Database& Database::instance()
{
  Database* db = instance.load(boost::memory_order_consume);
  if (!db)
  {
    boost::mutex::scoped_lock lock(mtx);
    db = instance.load(boost::memory_order_consume);
    if (!db)
    {
      db = new Database();
      instance.store(db, boost::memory_order_release);
    }
  }
}
```

由于本书关注的是现代 C++，我们不会进一步详细讨论这种方法。

## The Trouble with Singleton

假设我们的数据库包含了一张首都城市及其人口的列表。我们的单例数据库将要符合的接口是：

```c++
class Database
{
public:
  virtual int get_population(const std::string& name) = 0;
};
```

我们有一个成员函数，可以根据给定的城市获取其人口。现在，假设这个接口由一个名为 `SingletonDatabase` 的具体实现来采用，该实现以我们之前讨论的方式实现了单例模式：

```c++
class SingletonDatabase : public Database
{
  SingletonDatabase() { /* read data from database */ }
  std::map<std::string, int> capitals;
public:
  SingletonDatabase(SingletonDatabase const&) = delete;
  void operator=(SingletonDatabase const&) = delete;

  static SingletonDatabase& get()
  {
    static SingletonDatabase db;
    return db;
  }

  int get_population(const std::string& name) override
  {
    return capitals[name];
  }
};
```

正如我们所提到的，像上述那样的单例真正的问题在于它们在其他组件中的使用。我的意思是：假设基于前面的例子，我们构建了一个用于计算多个不同城市总人口的组件：

```c++
struct SingletonRecordFinder
{
  int total_population(std::vector<std::string> names)
  {
    int result = 0;
    for (auto& name : names)
      result += SingletonDatabase::get().get_population(name);
    return result;
  }
};
```

问题是 `SingletonRecordFinder` 现在牢固地依赖于 `SingletonDatabase`。这为测试带来了问题：如果我们想要检查 `SingletonRecordFinder` 是否正确工作，我们就需要使用实际数据库中的数据，也就是说：

```c++
TEST(RecordFinderTests, SingletonTotalPopulationTest)
{
  SingletonRecordFinder rf;
  std::vector<std::string> names{ "Seoul", "Mexico City" };
  int tp = rf.total_population(names);
  EXPECT_EQ(17500000 + 17400000, tp);
}
```

但是，如果我们不想在测试中使用实际的数据库呢？如果我们想要使用其他一些虚拟组件代替呢？好吧，在我们当前的设计中，这是不可能的，而正是这种灵活性的缺乏导致了单例模式的弊端。

那么，我们可以做些什么呢？首先，我们需要停止显式地依赖 `SingletonDatabase`。由于我们所需要的只是一个实现了 `Database` 接口的对象，我们可以创建一个新的 `ConfigurableRecordFinder`，它允许我们配置数据来源：

```c++
struct ConfigurableRecordFinder
{
  explicit ConfigurableRecordFinder(Database& db)
    : db{db} {}

  int total_population(std::vector<std::string> names)
  {
    int result = 0;
    for (auto& name : names)
      result += db.get_population(name);
    return result;
  }

  Database& db;
};
```

我们现在使用 `db` 引用而不是显式地使用单例。这让我们可以为测试记录查找器专门创建一个虚拟数据库：

```c++
class DummyDatabase : public Database
{
  std::map<std::string, int> capitals;
public:
  DummyDatabase()
  {
    capitals["alpha"] = 1;
    capitals["beta"] = 2;
    capitals["gamma"] = 3;
  }

  int get_population(const std::string& name) override {
    return capitals[name];
  }
};
```

现在，我们可以重写我们的单元测试以利用这个 `DummyDatabase`：

```c++
TEST(RecordFinderTests, DummyTotalPopulationTest)
{
  DummyDatabase db{};
  ConfigurableRecordFinder rf{ db };
  EXPECT_EQ(4, rf.total_population(std::vector<std::string>{"alpha", "gamma"}));
}
```

这个测试更加健壮，因为如果实际数据库中的数据发生变化，我们不必调整单元测试的值——虚拟数据保持不变。

## Singletons and Inversion of Control

显式地将一个组件设计为单例的方法具有明显的侵入性，并且如果将来决定不再将该类视为单例，这种决策可能会带来高昂的修改成本。一种替代解决方案是采用一种约定，即不直接强制类的生命周期，而是将此功能外包给一个 IoC（控制反转）容器。

以下是使用 Boost.DI 依赖注入框架定义单例组件的样子：

```c++
auto injector = di::make_injector(
  di::bind<IFoo>.to<Foo>.in(di::singleton),
  // other configuration steps here
);
```

在上述代码中，我使用类型名称中的第一个大写字母 “I” 来表示接口类型。实际上，`di::bind` 这行代码的意思是，每当我们需要一个成员类型为 `IFoo` 的组件时，我们用 `Foo` 的单例实例来初始化该组件。

根据许多人的看法，在 DI（依赖注入）容器中使用单例是唯一社会可接受的单例使用方式。至少通过这种方法，如果您需要替换单例对象为其他东西，您可以在一个中心位置进行：即容器配置代码中。额外的好处是不会自己实现任何单例逻辑，从而防止可能的错误。哦，还有，我提到 Boost.DI 是线程安全的吗？

## Monostate

单态（Monostate）是单例模式的一种变体。它是一个表现得像单例的类，但看起来像是一个普通的类。

```c++
class Printer
{
  static int id;
public:
  int get_id() const { return id; }
  void set_id(int value) { id = value; }
};
```

您能看出来这里发生了什么吗？这个类看起来像是一个带有 `getter` 和 `setter` 的普通类，但实际上这些方法操作的是 `static` 数据！

这可能看起来是一个非常巧妙的技巧：您允许人们实例化 `Printer`，但它们实际上都引用相同的数据。然而，用户怎么才能知道这一点呢？用户可能会愉快地实例化两个打印机，为它们分配不同的 ID，并且当发现两个打印机实际上是完全一样的时候会感到非常惊讶！

单态（Monostate）方法在某种程度上是有效的，并具有一些优点。例如，它易于继承，可以利用多态性，而且它的生命周期相对明确定义（尽管有时你可能不希望它是这样）。它最大的优势在于，您可以对已经在系统中广泛使用的现有对象进行修补，使其以单态方式行为，只要您的系统能够很好地处理对象实例的非复数性，那么您就可以获得一种类似单例的实现，而无需重写额外的代码。

缺点也是显而易见的：这是一种侵入性的方法（将普通对象转换为单态并不容易），并且由于使用了静态成员，即使不需要时它也始终占用空间。最终，单态的最大问题是它假设类字段总是通过 `getter` 和 `setter` 来暴露。如果字段被直接访问，那么您的重构几乎注定要失败。

## Summary

单例并不是完全邪恶的，但当它们被不加考虑地使用时，会破坏应用程序的可测试性和可重构性。如果您确实必须使用单例，尽量避免直接使用（例如，编写 `SomeComponent.getInstance().foo()`），而是应将其作为依赖项保持指定（例如，作为构造函数参数），并且所有依赖项都从应用程序中的单个位置（例如，控制反转容器）得到满足。
