# Observer

观察者模式是一种流行且必要的设计模式，因此令人惊讶的是，不像其他语言（例如 C#），C++ 以及标准库并没有提供现成的实现。尽管如此，一个安全且正确实现的观察者（如果真的能做到这一点的话）是一个技术上非常复杂的构建，因此在这一章中，我们将详细探讨它，包括所有复杂的技术细节。

## Property Observers

人们会变老，这是生活的事实。但当某人又长了一岁，我们可能想要在他们生日时向他们表示祝贺。但是，如何实现呢？鉴于这样的定义：

```c++
struct Person
{
  int age;
  Person(int age) : age{age} {}
};
```

我们如何知道一个人的年龄发生了变化？实际上，我们并不知道。为了看到变化，我们可以尝试轮询：每 100 毫秒读取一次人的年龄，并将新值与前一个值进行比较。这种方法虽然可行，但既繁琐又不具备良好的扩展性。我们需要更聪明的方法。

我们知道，我们希望在每次写入一个人的年龄字段时都能得到通知。那么，唯一能捕捉到这一点的方法是创建一个 `setter` 方法，也就是说：

```c++
struct Person
{
  int get_age() const { return age; }
  void set_age(const int value) { age = value; }
private:
  int age;
};
```

`set_age()` 方法是我们可以通知关心的人年龄确实已经改变的地方。但是，如何实现呢？

## Observer\<T\>

一种方法是定义一个基类，任何对获取 `Person` 的变化感兴趣的类都需要继承这个基类：

```c++
struct PersonListener
{
  virtual void person_changed(Person& p,
    const string& property_name, const any new_value) = 0;
};
```

然而，这种方法相当局限，因为属性变化可能发生在不仅仅是 `Person` 的其他类型上，我们并不希望为那些类型也创建额外的类。这里有一个更通用的方法：

```c++
template<typename T>
struct Observer
{
  virtual void field_changed(T& source, const string& field_name) = 0;
};
```

`field_changed()` 方法中的两个参数，希望是不言自明的。第一个参数是对实际发生字段变化的对象的引用，第二个参数是字段的名称。是的，名称是以 `string` 形式传递的，这确实会影响我们代码的可重构性（如果字段名称改变了怎么办？）。

这种实现将允许我们观察 `Person` 类的变化，并例如将这些变化写入命令行：

```c++
struct ConsolePersonObserver : Observer<Person>
{
  void field_changed(Person& source, const string& field_name) override
  {
    cout << "Person's " << field_name << " has changed to "
         << source.get_age() << ".\n";
  }
};
```

我们引入的灵活性使得我们可以观察多个类的属性变化。例如，如果我们添加一个 `Creature` 类，我们现在可以同时观察 `Person` 和 `Creature` 类的属性变化：

```c++
struct ConsolePersonObserver : Observer<Person>, Observer<Creature>
{
  void field_changed(Person& source, ...) { ... };
  void field_changed(Creature& source, ...) { ... };
}
```

另一种替代方案是使用 `std::any` 并摒弃泛型实现。试试看！

## Observable\<T\>

所以，让我们回到 `Person` 类的话题。既然它即将成为一个可观察的类，它将不得不承担新的职责，具体包括：

- 维护一个私有的观察者列表：保存所有对 `Person` 的变化感兴趣的观察者。
- 允许观察者订阅（subscribe()）/取消订阅（unsubscribe()）：使观察者可以注册或注销以接收 `Person` 的变化通知。
- 当实际发生更改时通知所有观察者：通过 `notify()` 方法告知所有观察者。

所有这些功能可以很轻松地移到一个单独的基类中，从而避免为每一个潜在的可观察类重复实现这些功能。

```c++
template <typename T>
struct Observable
{
  void notify(T& source, const string& name) { ... }
  void subscribe(Observer<T>* f) { observers.push_back(f); }
  void unsubscribe(Observer<T>* f) { ... }
private:
  vector<Observer<T>*> observers;
};
```

我们已经实现了 `subscribe()`——它只是将一个新的观察者添加到私有的观察者列表中。观察者列表不对任何人开放——甚至不对派生类开放。我们不希望人们随意操纵这个集合。

接下来，我们需要实现 `notify()`。想法很简单：遍历每一个观察者，并依次调用它们的 `field_changed()` 方法：

```c++
void notify(T& source, const string& name)
{
  for (auto obs : observers)
    obs->field_changed(source, name);
}
```

虽然继承自 `Observable<T>` 是必要的，但这还不够：我们的类还需要在任何字段更改时负责调用 `notify()`。

以 `set_age()` 为例，它现在有三个职责：

- 检查年龄是否确实发生了变化。如果年龄是 20 并且我们正在给它赋值 20，则没有必要执行任何赋值或通知。
- 将字段赋值为适当的值。
- 使用正确的参数调用 `notify()`。

因此，`set_age()` 的新实现将会如下所示：

```c++
struct Person : Observable<Person>
{
  void set_age(const int age)
  {
    if (this->age == age) return ;
    this->age = age;
    notify(*this, "age");
  }
private:
  int age;
};
```

## Connecting Observers and Observables

我们现在准备好开始使用我们创建的基础设施，以便在 `Person` 的字段发生变化时（其实，我们可以称之为属性）接收通知。以下是我们观察者的回顾：

```c++
struct ConsolePersonObserver : Observer<Person>
{
  void field_changed(Person& source, const string& field_name) override
  {
    cout << "Person's " << field_name << " has changed to "
         << source.get_age() << ".\n";
  }
};
```

以下是使用该基础设施的方式：

```c++
Person p{ 20 };
ConsolePersonObserver cpo;
p.suscribe(&cpo);
p.set_age(21); // Person's age has changed to 21.
p.set_age(22); // Person's age has changed to 22.
```

只要你不去关心属性依赖和线程安全 / 可重入性的问题，你可以在这里停下来，采用这个实现，并开始使用它。如果你想了解更复杂方法的讨论，请继续阅读。

## Dependency Problems

16 岁或以上的人（在你的国家可能有所不同）可以投票。因此，假设我们想要在一个人的投票权发生变化时收到通知。首先，假设我们的 `Person` 类型具有以下 getter：

```c++
bool get_can_vote() const { return age >= 16; }
```

请注意，`get_can_vote()` 没有支持字段（backing field）也没有 `setter`（我们可以引入这样的字段，但这显然是多余的），但我们仍然觉得有必要在它上面调用 `notify()`。但是，如何实现呢？好吧，我们可以尝试找出是什么导致了 `can_vote` 的变化……没错，是 `set_age()` 方法！因此，如果我们希望在投票状态发生变化时收到通知，这些通知需要在 `set_age()` 中进行。准备好，你将会有一个惊喜！

```c++
void set_age(int value) const
{
  if (age == value) return;

  auto old_can_vote = can_vote(); // store old value
  age = value;
  notify(*this, "age");

  if (old_can_vote != can_vote()) // check value has changed
    notify(*this, "can_vote");
}
```

前面的函数包含了太多内容。我们不仅检查 `age` 是否发生了变化，还检查 `can_vote` 是否发生变化，并且也对其进行了通知！你可能已经猜到这种方法扩展性不好，对吧？想象一下如果 `can_vote` 依赖于两个字段，比如 `age` 和 `citizenship`——这意味着它们的 `setter` 方法都必须处理 `can_vote` 的通知。而如果年龄还以这种方式影响其他十个属性呢？这是一个不可行的解决方案，会导致代码变得脆弱且难以维护，因为变量之间的关系需要手动跟踪。

简而言之，在上述场景中，`can_vote` 是 `age` 的一个 *dependent property*（依赖属性）。依赖属性的挑战本质上与 Excel 等工具面临的挑战相同：给定许多不同单元格之间的依赖关系，当其中一个单元格发生变化时，如何知道应该重新计算哪些单元格。

当然，属性依赖可以被形式化为某种 `map<string, vector<string>>`，用来保存受某个属性影响的属性列表（或者反过来，所有影响某个属性的属性）。遗憾的是，这个映射必须手动定义，并且保持其与实际代码同步相当棘手。

## Unsubscription and Thread Safety

有一件事我忽略了讨论，即观察者如何 `unsubscribe()`（取消订阅）一个可观察对象。通常情况下，你想要把自己从观察者列表中移除，在单线程场景下这很简单：

```c++
void unsubscribe(Observer<T>* observer)
{
  observer.earse(
    remove(observers.begin(), observers.end(), observer),
    observers.end());
};
```

虽然使用 `erase-remove` 惯用法在技术上是正确的，但它只适用于单线程场景。`std::vector` 不是线程安全的，因此同时调用 `subscribe()` 和 `unsubscribe()` 可能会导致意外的结果，因为这两个函数都会修改 `vector`。

这个问题很容易解决：只需在所有可观察对象的操作上加上锁即可。这可以像下面这样简单：

```c++
template <typename T>
struct Observable
{
  void notify(T& source, const string& name)
  {
      scope_lock<mutex> lock{ mtx };
      ...
  }
  void subscribe(Observer<T>* f)
  {
      scope_lock<mutex> lock{ mtx };
      ...
  }
  void unsubscribe(Observer<T>* o)
  {
      scope_lock<mutex> lock{ mtx };
      ...
  }
private:
  vector<Observer<T>*> observers;
  mutex mtx;
};
```

另一个非常可行的替代方案是使用类似 TPL / PPL 中的 `concurrent_vector`。当然，你会失去顺序保证（换句话说，依次添加两个对象并不能保证它们会按该顺序被通知），但它确实可以让你免于自己管理锁。

## Reentrancy

最后一个实现通过在需要时锁定三个关键方法中的任何一个来提供一定的线程安全性。但现在让我们想象以下场景：你有一个 `TrafficAdministration` 组件，它会持续监控一个人直到他们达到足够开车的年龄。当他们 17 岁时，该组件取消订阅：

```c++
struct TrafficAdministration : Observer<Person>
{
  void TrafficAdministration::field_changed(Person& source, const string& filed_name) override
  {
    if (field_name == "age")
    {
      if (source.get_age() < 17)
        cout << "Whoa there, you are not old enough to drive!\n";
      else
      {
        // oh, ok, they are old enough, let's not monitor them anymore
        cout << "We no longer care!\n";
        source.unsubscribe(this);
      }
    }
  }
};
```

这是一个问题，因为当年龄变为 17 岁时，整体的调用链将会是：

```c++
notify() --> field_changed() --> unsubscribe()
```

这是一个问题，因为在 `unsubscribe()` 中我们最终会尝试获取一个已经被占用的锁。这是一个 *reentrancy*（可重入性）问题。有几种不同的方法可以处理这个问题：

- 一种方法是简单地禁止这种情况。毕竟，在这个特定的例子中，可重入性非常明显。
- 另一种方法是放弃从集合中移除元素的想法。相反，我们可以选择如下方式：

```c++
void unsubscribe(Observer<T>* o)
{
  auto it != find(observers.begin(), observers.end(), o);
  if (it != observers.end())
    *it = nullptr;  // cannot do this for a set
}
```

随后，在你调用 `notify()` 时，你只需要添加一个额外的检查：

```c++
void notify(T& source, const string& name)
{
  for (auto obs : observers)
  {
    if (obs)
      obs->field_changed(source, name);
  }
}
```

当然，前述内容仅解决了 `notify()` 和 `subscribe()` 之间可能的争用问题。例如，如果你同时进行 `subscribe()` 和 `unsubscribe()`，这仍然是对集合的并发修改——并且它仍然可能会失败。因此，至少你可能想要在这里保持一个锁。

另一种可能性是在 `notify()` 中直接复制整个集合。你仍然需要锁，只是不将锁应用到通知上。我的意思是：

```c++
void notify(T& source, const string& name)
{
  vector<Observer<T>*> observers_copy;
  {
    lock_guard<mutex_t> lock{ mtx };
    observers_copy = observers;
  }
  for (auto obs : observers_copy)
    if (obs)
      obs->field_changed(source, name);
}
```

在前述实现中，我们确实获取了一个锁，但是在调用 `field_changed` 时，锁已经被释放了，因为它只在用于复制 `vector` 的人工作用域中创建。我不会担心这里的效率问题，因为指针的 `vector` 并不会占用太多内存。

最后，总是可以用 `recursive_mutex` 替换 `mutex`。一般来说，递归互斥锁被大多数开发者所诟病（Stack Overflow 上有证明），不仅因为性能上的影响，更多的是因为在大多数情况下（就像在这个观察者例子中一样），如果你稍微改进一下代码设计，就可以使用普通的、非递归的变体来解决问题。

这里还有一些我们没有真正讨论过的有趣的实际问题。它们包括以下几点：

- 如果同一个观察者被添加两次会发生什么？
- 如果我允许重复的观察者，`unsubscribe()` 是否会移除每一个实例？
- 如果我们使用不同的容器，行为会受到怎样的影响？例如，如果我们决定通过使用 `std::set` 或 `boost::unordered_set` 来防止重复，这对普通操作意味着什么？
- 如果我希望观察者根据优先级排序呢？

所有这些实际问题在基础稳固的情况下都是可以管理的。我们在这里不会进一步讨论这些问题。

## Observer via Boost.Signals2

有许多预封装的观察者模式实现，最著名的可能是 Boost.Signals2 库。本质上，这个库提供了一种称为 `signal` 的类型，在 C++ 术语中表示信号（在其他地方称为事件）。这个信号可以通过提供函数或 lambda 表达式来订阅。它也可以取消订阅，并且当你想要通知时，可以触发它。

使用 Boost.Signals2，我们可以如下定义 `Observer<T>`：

```c++
template <typename T>
struct Observable
{
  signal<void(T&, const string&)> property_changed;
};
```

以及它的调用看起来如下：

```c++
struct Person : Observable<Person>
{
  ...
  void set_age(const int age)
  {
    if (this->age == age) return;

    this->age = age;
    property_changed(*this, "age");
  }
};
```
实际使用该 API 时会直接使用信号，除非你决定添加更多的 API 封装以使其更易于使用：

```c++
Person p{123};
auto conn = p.property_changed.connect([](Person&, const string& prop_name)
{
  cout << prop_name << " has been changed" << endl;
});
p.set_age(20);  // name has been changed

// later, optionally
conn.disconnect();
```

`connect()` 调用的结果是一个连接对象，该对象也可以在你不再需要从信号接收通知时用于取消订阅。

## 总结

毫无疑问，本章中展示的代码是一个将问题思考和设计得远超大多数人期望的典型案例。

让我们回顾一下在实现观察者模式时的主要设计决策：

- 决定你希望你的可观察对象传达什么信息。例如，如果你正在处理字段/属性变化，你可以包括属性的名称。你还可以指定旧值和新值，但传递类型可能会有问题。
- 你希望观察者是完整的类，还是仅仅有一系列虚函数就足够了？
- 你希望如何处理观察者的取消订阅：
  - 如果你不打算支持取消订阅——恭喜，你在实现观察者模式时将省去很多努力，因为在可重入场景中没有移除问题。
  - 如果你计划支持显式的 `unsubscribe()` 函数，你可能不希望直接在函数中使用 `erase-remove`，而是标记元素以便稍后移除。
  - 如果你不喜欢通过（可能是空的）裸指针调度的想法，考虑使用 `weak_ptr` 代替。
- `Observer<T>` 的函数是否可能从多个不同的线程调用？如果是这样，你需要保护你的订阅列表：
  - 你可以在所有相关函数上放置一个 `scoped_lock`；或者
  - 你可以使用线程安全的集合，如 TBB/PPL 中的 `concurrent_vector`。你会失去顺序保证。
- 是否允许来自同一来源的多次订阅？如果允许，则不能使用 `std::set`。

遗憾的是，没有一个理想的观察者模式实现可以满足所有的需求。无论你选择哪种实现方式，都预计会有一些妥协。
