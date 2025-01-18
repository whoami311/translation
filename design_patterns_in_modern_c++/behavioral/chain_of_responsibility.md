# Chain of Responsibility

考虑一下公司不当行为的典型例子：内幕交易。假设某位交易员被当场抓住利用内部信息进行交易。谁应该为此负责？如果管理层不知情，那么责任在于该交易员。但如果交易员的同事也参与其中，那么小组经理可能是责任人。或者，这种行为可能在整个机构普遍存在，在这种情况下，首席执行官（CEO）将承担责任。

这是一个责任链模式的例子：您有系统中的多个不同元素，它们可以一个接一个地处理消息。作为一种概念，它相当容易实现，因为所隐含的只是使用某种类型的列表。

## Scenario

想象一个电脑游戏，其中每个生物都有一个名字和两个特征值——攻击和防御：

```c++
struct Creature
{
  string name;
  int attack, defense;
  // constructor and << here
};
```

现在，随着生物在游戏中不断前进，它可能会遇到一个物品（例如，一把魔法剑），或者它可能会被施加魔法。在这两种情况下，其攻击和防御值将由我们称之为 `CreatureModifier` 的东西进行修改。

此外，同时应用多个修饰器的情况并不少见，因此我们需要能够将修饰器堆叠在生物上，并允许它们按照附加的顺序依次应用。

让我们看看如何实现这一点。

## Pointer Chain

按照经典的职责链（Chain of Responsibility, CoR）模式，我们将如下实现 `CreatureModifier`：

```c++
class CreatureModifier
{
  class CreatureModifier* next{nullptr};
protected:
  Creature& creature; // alternative: pointer or shared_ptr
public:
  explicit CreatureModifier(Creature& creature)
    : creature(creature) {}

  void add(CreatureModifier* cm)
  {
    if (next)
      next->add(cm);
    else
      next = cm;
  }

  virtual void handle()
  {
    if (next) next->handle(); // critical!
  }
};
```

这里发生了很多事情，所以我们逐一讨论：

- 该类接收并存储一个它计划修改的 `Creature` 的引用。
- 该类本身并没有做太多事情，但它并不是抽象类：所有成员都有实现。
- `next` 成员指向紧跟在此之后的可选 `CreatureModifier`。这意味着，所指向的修饰器是 `CreatureModifier` 的继承者。
- `add()` 函数将另一个生物修饰器添加到修饰器链中。这是递归完成的：如果当前修饰器是 `nullptr`，我们就将其设置为新的修饰器；否则，我们遍历整个链并将新的修饰器放在末尾。
- `handle()` 函数只是简单地处理链中的下一个项目（如果存在的话）；它本身没有行为。它是虚函数这一事实意味着它是为了被重写而设计的。

到目前为止，我们所拥有的只是一个简单的追加操作的单向链表的实现。但当我们开始从它继承时，情况应该会变得更加清晰。例如，以下是如何创建一个将生物攻击值加倍的修饰器的方法：

```c++
class DoubleAttackModifier : public CreatureModifier
{
public:
  explicit DoubleAttackModifier(Creature& creature)
    : CreatureModifier(creature) {}

  void handle() override
  {
    creature.attack *= 2;
    CreatureModifier::handle();
  }
};
```

好的，我们终于有所进展了。所以这个修饰器继承自 `CreatureModifier`，并在其 `handle()` 方法中做了两件事：将攻击值翻倍，并调用基类的 `handle()` 方法。第二部分是关键的：修饰器链能够应用的唯一方式是每个继承者在自己的 `handle()` 实现结束时不要忘记调用基类的方法。

下面是一个更复杂的修饰器。这个修饰器会增加攻击值为 2 或更低的生物的防御值 1 点：

```c++
class IncreaseDefenseModifier : public CreatureModifier
{
public:
  explicit IncreaseDefenseModifier(Creature& creature)
    : CreatureModifier(creature) {}

  void handle() override
  {
    if (creature.attack <= 2)
      creature.defense += 1;
    CreatureModifier::handle();
  }
};
```

再次强调，我们在最后调用了基类的方法。将所有这些放在一起，我们现在可以创建一个生物，并应用一系列修饰器到它身上：

```c++
Creature goblin{ "Goblin", 1, 1 };
CreatureModifier root{ goblin };
DoubleAttackModifier r1{ goblin };
DoubleAttackModifier r1_2{ goblin };
IncreaseDefenseModifier r2{ goblin };

root.add(&r1);
root.add(&r1_2);
root.add(&r2);

root.handle();

cout << goblin << endl;
// name: Goblin attack: 4 defense: 1
```

如您所见，哥布林的属性是 4/1，因为它的攻击值被翻倍了两次，而虽然添加了防御修饰器，但并未影响其防御分数。

这里还有一个有趣的问题。假设您决定对生物施放一个咒语，使得任何增益效果都无法应用到它身上。这样做容易吗？实际上非常简单，因为您只需要避免调用基类的 `handle()` 方法：这样就可以阻止整个链的执行。

```c++
class NoBonusesModifier : public CreatureModifier
{
public:
  explicit NoBonusesModifier(Creature& creature)
    : CreatureModifier(creature) {}

  void handle() override
  {
    // nothing here!
  }
};
```

就是这样！现在，如果您将 `NoBonusesModifier` 插入链的开头，那么后续的修饰器将不会被应用。

## Broker Chain

指针链的例子非常人为。在现实世界中，您希望生物能够任意地获得和失去增益效果，而追加操作的单向链表并不支持这种灵活性。此外，您不希望永久修改底层生物的状态（如我们之前所做的），而是希望保持这些修改是临时的。

一种实现职责链模式（CoR）的方法是通过一个集中化的组件。这个组件可以维护游戏中所有修饰器的列表，并通过确保所有相关的增益都被应用来促进对特定生物攻击或防御的查询。

我们将要构建的组件被称为事件代理（event broker）。由于它连接到每个参与的组件，因此它代表了中介者（Mediator）设计模式；并且因为它通过事件响应查询，所以它还利用了观察者（Observer）设计模式。

现在让我们开始构建。首先，我们将定义一个名为 `Game` 的结构，该结构将表示正在玩的游戏：

```c++
struct Game // mediator
{
  signal<void(Query&)> queries;
};
```

我们使用 Boost.Signals2 库来维护一个名为 `queries` 的信号。这实际上允许我们触发这个信号，并由每个槽（监听组件）来处理它。但是，事件与查询生物的攻击或防御有什么关系呢？

好吧，想象一下您想要查询一个生物的统计数据。您当然可以尝试直接读取字段，但请记住：我们需要在最终值确定之前应用所有的修饰器。因此，我们将把查询封装在一个单独的对象中（这是命令模式），其定义如下：

```c++
struct Query
{
  string creature_name;
  enum Argument { attack, defense } argument;
  int result;
};
```

在前述的类中，我们所做的只是封装了从生物查询特定值的概念。我们只需要提供生物的名字以及我们感兴趣的统计数据。正是这个值（确切地说，是它的引用）将由 `Game::queries` 构建和使用，以应用修饰器并返回最终的值。

现在，让我们继续定义 `Creature` 类。它与之前非常相似。字段方面唯一的不同是添加了一个对 `Game` 的引用。

```c++
class Creature
{
  Game& game;
  int attack, defense;
public:
  string name;
  Creature(Game& game, ...) : game{game}, ... { ... }
  // other members here
};
```

现在，请注意 `attack` 和 `defense` 是 `private`。这意味着，要获取最终的（经过修饰器处理后的）攻击值，您需要调用一个单独的 `getter` 函数，例如：

```c++
int Creature::get_attack() const
{
  Query q{ name, Query::Argument::attack, attack };
  game.queries(q);
  return q.result;
}
```

这就是魔法发生的地方！我们并不是简单地返回一个值或静态地应用某个基于指针的链，而是创建一个带有正确参数的 `Query`，然后将这个查询发送给订阅了 `Game::queries` 的任何组件进行处理。每个订阅的组件都有机会修改基础攻击值。

因此，现在让我们来实现修饰器。我们将再次创建一个基类，但这次它不会有一个 `handle()` 方法：

```c++
class CreatureModifier
{
  Game& game;
  Creature& creature;
public:
  CreatureModifier(Game& game, Creature& creature)
    : game(game), creature(creature) {}
};
```

因此，修饰器的基类并不是特别有趣。实际上，您可以完全不用它，因为它唯一的作用就是确保构造函数被正确地调用并传入正确的参数。但既然我们选择了这种方法，现在让我们继承 `CreatureModifier` 并看看如何执行实际的修改：

```c++
class DoubleAttackModifier : public CreatureModifier
{
  connection conn;
public:
  DoubleAttackModifier(Game& game, Creature& creature)
    : CreatureModifier(game, creature)
  {
    conn = game.queries.connect([&](Query& q)
    {
       if (q.creature_name == creature.name &&
         q.argument == Query::Argument::attack)
         q.result *= 2;
     });
   }

   ~DoubleAttackModifier() { conn.disconnect(); }
};
```

如您所见，所有的操作都在构造函数和析构函数中完成；不需要额外的方法。在构造函数中，我们使用 `Game` 的引用获取 `Game::queries` 信号并连接到它，指定一个使攻击值翻倍的 lambda 函数。当然，lambda 必须做一些检查：我们需要确保增强的是正确的生物（通过名称比较），并且我们所关心的统计数据确实是攻击。这两项信息都保存在 `Query` 的引用中，以及我们要修改的初始 `result` 值。

我们还注意存储信号连接，以便在对象被销毁时断开连接。这样，我们可以临时应用修饰器，并让它在修饰器超出作用域时自动失效。

将所有这些放在一起，我们得到如下内容：

```c++
Game game;
Creature goblin{ game, "Strong Goblin", 2, 2 };
cout << goblin << endl;
// name: Strong Goblin attack: 2 defense: 2
{
  DoubleAttackModifier dam{ game, goblin };
  cout << goblin << endl;
  // name: Strong Goblin attack: 4 defense: 2
}
cout << goblin << endl;
// name: Strong Goblin attack: 2 defense: 2
}
```

这里发生了什么？嗯，在被修改之前，哥布林的属性是 2/2。然后，我们制造了一个作用域，在这个作用域内，哥布林受到 `DoubleAttackModifier` 的影响，所以在作用域内它变成了一个 4/2 的生物。一旦我们退出作用域，修饰器的析构函数就会触发，并且它会从代理中断开连接，因此在查询值时不再对其产生影响。因此，哥布林的属性在退出作用域后又变回了 2/2。

## Summary

职责链（Chain of Responsibility, CoR）是一种非常简单的设计模式，它允许组件依次处理命令（或查询）。最简单的 CoR 实现方式是创建一个指针链，并且理论上你可以用一个普通的 `vector` 来替代它，或者如果你想实现快速移除，也可以使用列表 `list`。

一种更为复杂的代理链（Broker Chain）实现不仅利用了中介者（Mediator）和观察者（Observer）模式，还允许我们在事件（信号）上处理查询。这使得每个订阅者可以在最终值返回给客户端之前，对传递的对象进行修改（这是一个在整个链中传递的单一引用）。
