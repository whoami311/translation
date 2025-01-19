# Command

考虑一个简单的变量赋值，比如 `meaning_of_life = 42`。变量确实被赋值了，但没有任何记录表明赋值发生了。没有人能告诉我们之前的值是什么。我们无法将赋值的事实序列化到某个地方。这是有问题的，因为没有变更记录，我们就无法回滚到之前的值、执行审计或进行基于历史的调试。

命令（Command）设计模式提出了一种不同的方法：与其直接通过对象的 API 来操作它们，不如向它们发送命令——即如何执行某项操作的指令。命令不过是一个数据类，其成员描述了要做什么以及如何做。让我们来看一个典型的场景。

## Scenario

让我们尝试建模一个典型的银行账户，该账户具有余额和透支限额。我们将在其上实现 `deposit()` 和 `withdraw()` 函数：

```c++
struct BankAccount
{
  int balance = 0;
  int overdraft_limit = -500;

  void deposit(int amount) {
    balance += amount;
    cout << "deposited " << amount << ", balance is now " << balance << "\n";
  }

  void withdraw(int amount)
  {
    if (balance - amount >= overdraft_limit)
    {
      balance -= amount;
      cout << "withdrew " << amount << ", balance is now " << balance << "\n";
    }
  }
};
```

现在我们当然可以直接调用成员函数，但假设为了审计的目的，我们需要记录每一次的存款和取款，并且我们不能直接在 `BankAccount` 类内部实现这一点，因为——您可能已经猜到了——我们已经设计、实现并测试了这个类。

##  Implementing the Command Pattern

我们将首先定义一个命令的接口：

```c++
struct Command
{
  virtual void call() const = 0;
};
```

定义好接口之后，我们现在可以使用它来定义一个 `BankAccountCommand`，该命令将封装关于如何操作银行账户的信息：

```c++
struct BankAccountCommand : Command
{
  BankAccount& account;
  enum Action { deposit, withdraw } action;
  int amount;

   BankAccountCommand(BankAccount& account, const Action action, const int amount)
    : account(account), action(action), amount(amount) {}
```

命令中包含的信息包括以下内容：

- 要操作的账户
- 要采取的操作；选项集和存储这些选项的变量在单个声明中定义
- 要存入或取出的金额

一旦客户端提供了这些信息，我们就可以使用它们来执行存款或取款操作：

```c++
void call() const override
{
  switch (action)
  {
  case deposit:
    account.deposit(amount);
    break;
  case withdraw:
    account.withdraw(amount);
    break;
  }
}
```

采用这种方法，我们可以创建命令，然后直接在命令上执行账户的修改：

```c++
BankAccount ba;
Command cmd{ba, BankAccountCommand::deposit, 100};
cmd.call();
```

这将会向我们的账户存入 100 美元。很简单！而且，如果您担心我们仍然向客户端暴露了原始的 `deposit()` 和 `withdraw()` 成员函数，您可以将它们设为 `private`，并简单地将 `BankAccountCommand` 设为友元类。

## Undo Operations

由于命令封装了对 `BankAccount` 的所有修改信息，因此它同样可以回滚这些修改，并将其目标对象恢复到之前的状态。

首先，我们需要决定是否将与撤销（undo）相关的操作加入到我们的 `Command` 接口中。为了简洁起见，我在这里会这样做，但通常来说，这是一个需要尊重接口分隔原则（Interface Segregation Principle）的设计决策，这个原则我们在本书开头（第一章）讨论过。例如，如果您设想某些命令是最终的且不受撤销机制影响，那么将 `Command` 接口拆分为比如 `Callable` 和 `Undoable` 可能是有意义的。

无论如何，这里是更新后的 `Command` 接口；请注意，我故意从函数中移除了 `const`：

```c++
struct Command
{
  virtual void call() = 0;
  virtual void undo() = 0;
};
```

以下是一个天真的 `BankAccountCommand::undo()` 实现，它基于（错误的）假设，即账户存款和取款是对称操作：

```c++
void undo() override
{
  switch (action)
  {
  case withdraw:
    account.deposit(amount);
    break;
  case deposit:
    account.withdraw(amount);
    break;
  }
}
```

为什么这个实现是错误的？因为如果你尝试取款的金额等同于一个发达国家的 GDP，你将不会成功，但在回滚交易时，我们没有方法知道它失败了！

为了获取这些信息，我们将修改 `withdraw()` 方法，使其返回一个成功标志：

```c++
bool withdraw(int amount)
{
  if (balance - amount >= overdraft_limit)
  {
    balance -= amount;
    cout << "withdrew " << amount << ", balance now " <<
      balance << "\n";
    return true;
  }
   return false;
}
```

这要好得多！我们现在可以修改整个 `BankAccountCommand` 以实现两个功能：

- 在进行取款时，内部存储一个成功标志。
- 在调用 `undo()` 时使用这个标志。

```c++
struct BankAccountCommand : Command
{
  ...
  bool withdrawal_succeeded;

  BankAccountCommand(BankAccount& account, const Action action,
    const int amount)
    : ..., withdrawal_succeeded{false} {}

  void call() override
  {
    switch (action)
    {
      ...
    case withdraw:
      withdrawal_succeeded = account.withdraw(amount);
      break;
    }
  }
```

现在您应该明白为什么我从 `Command` 的成员函数中移除了 `const` 了吧？既然我们现在分配了一个成员变量 `withdrawal_succeeded`，我们就不能再声称 `call()` 是常量成员函数（`const`）。我本可以保留 `undo()` 的 `const`，但这几乎没有好处。

好了，既然我们已经有了这个标志，我们可以改进 `undo()` 的实现了：

```c++
void undo() override
{
  switch (action)
  {
  case withdraw:
    if (withdrawal_succeeded)
      account.deposit(amount);
    break;
    ...
   }
}
```

终于，我们可以以一致的方式撤销取款命令了。

这次练习的目标当然是为了说明，除了存储要执行的操作信息之外，命令（Command）还可以存储一些中间信息，这些信息对于审计等用途非常有用：如果您检测到一系列 100 次失败的取款尝试，您可以调查潜在的安全入侵事件。

## Composite Command

从账户 A 向账户 B 转账可以通过两个命令来模拟：

1. Withdraw $X from A
2. Deposit $X to B

如果我们可以创建并调用一个单一的命令来封装这两个命令，而不是分别创建和调用这两个命令，那将是非常好的。这正是我们在第8章讨论的组合（Composite）设计模式的本质。

让我们定义一个组合命令的骨架。我将继承自 `vector<BankAccountCommand>`——这可能会有问题，因为 `std::vector` 没有虚析构函数，但在我们的情况下这不是问题。因此，这里是一个非常简单的定义：

```c++
struct CompositeBankAccountCommand : vector<BankAccountCommand>, Command
{
  CompositeBankAccountCommand(const initializer_list <value_type>& items)
    : vector<BankAccountCommand>(items) {}

  void call() override
  {
    for (auto& cmd : *this)
      cmd.call();
  }

  void undo() override
  {
    for (auto it = rbegin(); it != rend(); ++it)
      it->undo();
  }
};
```

如您所见，`CompositeBankAccountCommand` 同时也是一个 `vecotr` 和一个 `Command`，这符合组合（Composite）设计模式的定义。我添加了一个接收初始化列表的构造函数（非常有用！），并实现了 `undo()` 和 `redo()` 操作。请注意，`undo()` 过程是以相反的顺序遍历命令的；希望我不需要解释为什么这种默认行为是有必要的。

那么现在，对于专门用于转账的组合命令，我会这样定义它：

```c++
struct MoneyTransferCommand : CompositeBankAccountCommand
{
  MoneyTransferCommand(BankAccount& from,
    BankAccount& to, int amount) :
    CompositeBankAccountCommand
    {
      BankAccountCommand{from, BankAccountCommand::raw, amount},
      BankAccountCommand{to, BankAccountCommand::deposit, amount}
    } {}
};
```

如您所见，我们所做的只是重用基类的构造函数来初始化对象，并使用两个命令，然后重用基类的 `call()` 和 `undo()` 实现。

但是等等，这不对劲，是吗？基类的实现并不完全适用，因为它们没有包含失败的概念。如果我未能从账户 A 取款，那么我不应该将这笔钱存入账户 B；整个链路应当自行取消。

为了支持这一概念，需要进行更彻底的更改。我们需要：

- 在 `Command` 中添加一个成功标志。
- 记录每个操作的成功或失败情况。
- 确保只有当命令最初成功时，它才能被撤销。
- 引入一个新的中间类，称为 `DependentCompositeCommand`，它在回滚命令时非常谨慎。

在调用每个命令时，我们只会在前一个命令成功的情况下执行；否则，我们只是简单地将成功标志设置为 `false`。

```c++
void call() override
{
  bool ok = true;
  for (auto& cmd : *this)
  {
    if (ok)
    {
      cmd.call();
      ok = cmd.succeeded;
    }
    else
    {
      cmd.succeeded = false;
    }
  }
}
```

没有必要覆盖 `undo()`，因为每个命令都会检查自己的成功标志，并且只有当它被设置为 `true` 时才会撤销操作。

可以想象一个更为严格的形式，其中组合命令只有在其所有部分都成功时才视为成功（考虑一下转账的情况，取款成功但存款失败——您会希望这个交易通过吗？）——这实现起来要复杂一些，我再次将其作为留给读者的练习。

本节的全部目的是说明，基于命令（Command）的简单方法在考虑到现实世界的业务需求时可能会变得相当复杂。您是否真的需要这种复杂性……嗯，这取决于您自己决定。

##  Command Query Separation

命令查询分离（Command Query Separation, CQS）的概念是指系统中的操作大致可以分为以下两类：

- **命令（Commands）**，它们是指示系统执行某些涉及状态变更的操作的指令，但不返回任何值。
- **查询（Queries）**，它们是请求信息的指令，返回值但不改变状态。

任何当前直接暴露其状态以供读取和写入的对象都可以隐藏其状态（将其设为 `private`），然后，代替提供 `getter` 和 `setter` 对，它可以提供一个统一的接口。我的意思是：假设我们有一个具有名为 `strength` 和 `agility` 两个属性的生物（Creature）。我们可以这样定义这个生物：

```c++
class Creature
{
  int strength, agility;
public:
  Creature(int strength, int agility)
    : strength{strength}, agility{agility} {}

  void process_command(const CreatureCommand& cc);
  int process_query(const CreatureQuery& q) const;
};
```

如您所见，这里没有 `getter` 和 `setter`，但我们确实有两个 API 成员，分别叫做 `process_command()` 和 `process_query()`，它们用于所有与 `Creature` 对象的交互。这两个都是专用类，连同 `CreatureAbility` 枚举一起，定义如下：

```c++
enum class CreatureAbility { strength, agility };

struct CreatureCommand
{
  enum Action { set, increaseBy, decreaseBy } action;
  CreatureAbility ability;
  int amount;
};

struct CreatureQuery
{
  CreatureAbility ability;
};
```

如您所见，命令描述了您想要更改的成员以及如何更改它，包括更改的程度。查询对象仅指定要查询的内容，并且我们假设查询的结果是从函数返回的，而不是设置在查询对象本身中（如果其他对象影响这个对象，正如我们已经看到的，那就是替代的做法）。

因此，以下是 `process_command()` 的定义：

```c++
void Creature::process_command(const CreatureCommand &cc)
{
  int* ability;
  switch (cc.ability)
  {
    case CreatureAbility::strength:
      ability = &strength;
      break;
    case CreatureAbility::agility:
      ability = &agility;
      break;
  }
  switch (cc.action)
  {
    case CreatureCommand::set:
      *ability = cc.amount;
      break;
    case CreatureCommand::increaseBy:
      *ability += cc.amount;
      break;
    case CreatureCommand::decreaseBy:
      *ability -= cc.amount;
      break;
  }
};
```

以下是更为简单的 `process_query()` 定义：

```c++
int Creature::process_query(const CreatureQuery &q) const
{
  switch (q.ability)
  {
    case CreatureAbility::strength: return strength;
    case CreatureAbility::agility: return agility;
  }
  return 0;
}
```

如果您希望对这些命令和查询进行日志记录或持久化，现在您只需要在两个地方进行处理。这种方法唯一真正的问题是，对于那些只想以熟悉的方式操作对象的人来说，API 可能会难以使用。

幸运的是，如果我们想要的话，我们总是可以创建 `getter` / `setter` 对；这些方法将只是调用带有适当参数的 `process_` 方法：

```c++
void Creature::set_strength(int value)
{
  process_command(CreatureCommand{
    CreatureCommand::set, CreatureAbility::strength, value
  });
}

int Creature::get_strength() const
{
  return process_query(CreatureQuery{CreatureAbility::strength});
}
```

诚然，前述内容是对实际在实行命令查询分离（CQS）的系统内部发生的情况的一个非常简化的说明，但它希望能给出一个如何将所有对象接口拆分为命令（Command）和查询（Query）部分的概念。

## Summary

命令设计模式非常简单：它基本上建议的是，对象可以通过使用封装了指令的特殊对象来进行通信，而不是将这些相同的指令作为方法的参数来指定。

有时，您并不希望这样的对象改变目标对象或导致其执行特定操作；相反，您想要使用这样的对象从目标对象查询一个值，在这种情况下，我们通常称这样的对象为查询（Query）。在大多数情况下，查询是一个依赖于方法返回类型的不可变对象，但在某些情况下（例如，参见责任链代理链示例；第13章），您希望被返回的结果能够被其他组件修改。不过，组件本身并未被修改，只有结果被修改了。

命令在用户界面系统中被广泛用于封装典型动作（例如，复制或粘贴），然后允许通过多种不同的方式调用单个命令。例如，您可以使用顶级应用程序菜单、工具栏上的按钮或键盘快捷键来执行复制操作。最终，这些动作可以组合成宏——可以记录下来并根据需要重放的动作序列。
