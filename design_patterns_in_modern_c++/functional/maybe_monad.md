# Maybe Monad

在 C++ 中，如同在许多其他语言一样，我们有不同的方式来表达值的存在或缺失。具体到 C++，我们可以使用以下任何一种方法：

- 使用 `nullptr` 来编码缺失。
- 使用智能指针（例如 `std::shared_ptr`），它同样可以被测试是否存在值。
- `std::optional<T>` 是一个库解决方案；它可以存储类型为 `T` 的值，或者如果值缺失则存储 `std::nullopt`。

假设我们决定采用 `nullptr` 方法。在这种情况下，让我们想象一下，我们的域模型定义了一个 `Person`，该 `Person` 可能有也可能没有 `Address`，而 `Address` 又可以有一个可选的 `house_name`：

```c++
struct Address {
  string* house_name = nullptr;
};

struct Person {
  Address* address = nullptr;
};
```

我们感兴趣的是编写一个函数，给定一个人，如果存在的话，则安全地打印该人的房屋名称。在“传统” C++ 中，我们会这样实现：

```c++
void print_house_name(Person* p)
{
  if (p != nullptr &&
    p->address != nullptr &&
    p->address->house_name != nullptr) // ugh!
  cout << *p->address->house_name << endl;
}
```

前述代码表示了深入对象结构的过程，同时小心避免访问 `nullptr` 值。这种深入的过程可以通过使用 *Maybe Monad* 以函数式的方式表示。

为了构造这个单子，我们将定义一个新的类型 `Maybe<T>`。这个类型将被用作参与深入过程的临时对象：

```c++
template <typename T>
struct Maybe {
  T* context;
  Maybe(T *context) : context(context) { }
};
```

到目前为止，`Maybe` 看起来像是一个指针容器，没有什么特别令人兴奋的地方。它也不是非常实用，因为给定一个 `Person* p`，我们无法创建 `Maybe(p)`，这是由于我们无法从构造函数传递的参数中推导类模板参数。在这种情况下，我们还创建了一个辅助全局函数，因为函数实际上可以推导模板参数：

```c++
template <typename T>
Maybe<T> maybe(T* context)
{
  return Maybe<T>(context);
}
```

现在，我们想要做的是给 `Maybe` 添加一个成员函数，该函数：

- 如果 `context != nullptr`，则深入对象；
- 如果 `context` 实际上是 `nullptr`，则不执行任何操作。

“深入”的过程被封装进一个模板参数 `Func`，如下所示：

```c++
template <typename Func>
auto With(Func evaluator)
{
  return context != nullptr ? maybe(evaluator(context)) : nullptr;
}
```

前述内容是一个高阶函数的例子，也就是说，一个接受函数作为参数的函数。我们创建的这个函数接受另一个名为 `evaluator` 的函数，如果当前上下文非空，则可以在该上下文上调用 `evaluator` 并返回一个可以被包装在另一个 `Maybe` 中的指针。这种技巧允许 `With()` 调用的链式操作。

现在，以类似的方式，我们可以创建另一个成员函数，这次是直接在 `context` 上调用给定的函数，而不改变 `context` 本身：

```c++
template <typename TFunc>
auto Do(TFunc action)
{
  if (context != nullptr) action(context);
  return *this;
}
```

我们完成了！现在我们可以将 `print_house_name()` 函数重新定义为如下：

```c++
void print_house_name(Person* p)
{
  auto z = maybe(p)
    .With([](auto x) { return x->address; })
    .With([](auto x) { return x->house_name; })
    .Do([](auto x) { cout << *x << endl; });
}
```

这里有几个需要注意的地方。首先，我们成功创建了一个流畅接口，也就是说，函数调用可以一个接一个地链式调用。这是有道理的，因为每个操作符（如 `With`、`Do` 等）都返回要么是 `*this` 要么是一个新的 `Maybe<T>`。同样值得注意的是，在每一步中，深入的过程都是由一个 lambda 函数封装的。

如您可能猜测的那样，前述方法确实存在性能成本，尽管这些成本难以预测，并且取决于编译器优化代码的能力。这种方法也远非完美，因为我很希望能省略 `[](auto x)` 部分，转而使用某种简写符号。理想情况下，像 `maybe(p).With{it->address}` 这样的语法会很好。
