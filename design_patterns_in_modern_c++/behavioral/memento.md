# Memento

当我们研究命令（Command）设计模式时，我们注意到记录每一个单独的更改理论上允许你将系统回滚到任何时间点——毕竟，你保存了所有修改的记录。

然而，有时候，你并不真的关心回放系统的状态，但你确实希望能够在必要时将系统回滚到特定状态。

这正是备忘录（Memento）模式所做的：它存储系统的状态，并将其作为专用的、只读的对象返回，该对象自身没有任何行为。你可以把这个 “token” 仅用于将其反馈给系统以恢复到它所代表的状态。

让我们来看一个例子。

## Bank Account

让我们使用之前做过的银行账户的例子：

```c++
class BankAccount
{
  int balance = 0;
public:
  explicit BankAccount(const int balance)
    : balance(balance) {}
```

但现在我们决定创建一个只具有 `deposit()` 函数的银行账户。与之前的例子不同，`deposit()` 不再是 `void` 类型，而是被设计为返回一个 `Memento`：

```c++
Memento deposit(int amount)
{
  balance += amount;
  return { balance };
}
```

然后，这个 `Memento` 可以用于将账户回滚到之前的状态：

```c++
  void restore(const Memento& m)
  {
    balance = m.balance;
  }
};
```

至于 `Memento` 本身，我们可以采用一个简单的实现：

```c++
class Memento
{
  int balance;
public:
  Memento(int balance)
    : balance(balance)
  {
  }

  friend class BankAccount;
};
```

这里有两点需要注意：

- `Memento` 类是不可变的。想象一下，如果实际上可以更改余额：你可以将账户回滚到一个它从未存在过的状态！
- `Memento` 将 `BankAccount` 声明为友元类。这允许账户实际使用 `balance` 字段。另一种可行的替代方案是将 `Memento` 作为 `BankAccount` 的内部类。

以下是使用这种设置的方式：

```c++
void memento()
{
  BankAccount ba{ 100 };
  auto m1 = ba.deposit(50);
  auto m2 = ba.deposit(25);
  cout << ba << "\n"; // Balance: 175

  // undo to m1
  ba.restore(m1);
  cout << ba << "\n"; // Balance: 150

  // redo
  ba.restore(m2);
  cout << ba << "\n"; // Balance: 175
}
```

这个实现已经足够好，但确实缺少一些东西。例如，你永远无法获得一个代表初始余额的 `Memento`，因为构造函数不能返回值。你可以通过在构造函数中放置一个指针来解决这个问题，但这看起来有点不优雅。

## Undo and Redo

如果我们存储由 `BankAccount` 生成的每一个 `Memento`，那么我们将拥有一个类似于我们实现命令（Command）模式的情况，在这种情况下，撤销（undo）和重做（redo）操作是这种记录的副产品。让我们看看如何使用 `Memento` 来实现撤销/重做功能。

我们将引入一个新的银行账户类 `BankAccount2`，它将保存其生成的每一个 `Memento`： 

```c++
class BankAccount2 // supports undo/redo
{
  int balance = 0;
  vector<shared_ptr<Memento>> changes;
  int current;
public:
  explicit BankAccount2(const int balance) : balance(balance)
  {
    changes.emplace_back(make_shared<Memento>(balance));
    current = 0;
  }
```

我们现在解决了返回到初始余额的问题：初始更改的备忘录也被存储了。当然，这个备忘录实际上并没有被返回，所以为了回滚到它，我想你可以实现一些 `reset()` 函数之类的功能——这完全取决于你。

在前面的内容中，我们使用 `shared_ptr` 来存储备忘录，并且我们也使用 `shared_ptr` 来返回它们。此外，我们使用 `current` 字段作为更改列表中的一个 “pointer”，这样如果我们决定撤销并回退一步，我们总是可以重做并恢复到刚才的状态。

现在，这里是 `deposit()` 函数的实现：

```c++
shared_ptr<Memento> deposit(int amount)
{
  balance += amount;
  auto m = make_shared<Memento>(balance);
  changes.push_back(m);
  ++current;
  return m;
}
```

现在来谈谈有趣的部分（我们仍然在列出 `BankAccount2` 的成员）。我们添加一个基于备忘录恢复账户状态的方法：

```c++
void restore(const shared_ptr<Memento>& m)
{
  if (m)
  {
    balance = m->balance;
    changes.push_back(m);
    current = changes.size() - 1;
  }
}
```

恢复过程与我们之前看到的有显著不同。首先，我们实际上检查 `shared_ptr` 是否已初始化——这很重要，因为我们现在有一种方式来表示无操作（no-ops）：只需返回一个默认值即可。此外，当我们恢复一个备忘录时，我们实际上会将该备忘录推入更改列表中，这样撤销（undo）操作可以正确地作用于它。

现在，这里是 `undo()` 的实现（相当棘手）：

```c++
shared_ptr<Memento> undo()
{
  if (current > 0)
  {
    --current;
    auto m = changes[current];
    balance = m->balance;
    return m;
  }
  return{};
}
```

我们只有在 `current` 指向的更改大于零时才能调用 `undo()`。如果情况如此，我们将指针回退一步，获取该位置的更改，应用它，然后返回该更改。如果我们无法回滚到之前的备忘录，我们将返回一个默认构造的 `shared_ptr`，并在 `restore()` 中检查它。

`redo()` 的实现非常类似：

```c++
shared_ptr<Memento> redo()
{
  if (current + 1 < changes.size())
  {
    ++current;
    auto m = changes[current];
    balance = m->balance;
    return m;
  }
  return{};
}
```

同样，我们需要能够重做某件事：如果可以，我们安全地执行它；如果不可以，我们什么也不做并返回一个空指针。将所有这些放在一起，我们现在可以开始使用撤销 / 重做功能：

```c++
BankAccount2 ba{ 100 };
ba.deposit(50);
ba.deposit(25); // 125
cout << ba << "\n";

ba.undo();
cout << "Undo 1: " << ba << "\n"; // Undo 1: 150
ba.undo();
cout << "Undo 2: " << ba << "\n"; // Undo 2: 100
ba.redo();
cout << "Redo 2: " << ba << "\n"; // Redo 2: 150
ba.undo(); // back to 100 again
```

## 总结

备忘录（Memento）模式的核心在于分发可以用来将系统恢复到先前状态的令牌。通常，这个令牌包含将系统移动到特定状态所需的所有信息，如果它足够小，你也可以用它来记录系统的所有状态，从而不仅允许将系统任意重置为先前的状态，还支持对系统所有经历过的状态进行受控的向后（撤销，undo）和向前（重做，redo）导航。
