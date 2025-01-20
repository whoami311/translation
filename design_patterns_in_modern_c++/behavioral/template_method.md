# Template Method

策略模式和模板方法设计模式非常相似，以至于我可能会忍不住将这些模式合并为一个单一的 Skeleton Method（骨架方法）设计模式，就像工厂模式一样。不过，我会抵制这种冲动。

策略模式与模板方法的区别在于，策略模式使用组合（无论是静态还是动态），而模板方法使用继承。但定义算法骨架在一个地方、而其实现细节在其他地方定义的核心原则依然存在，并且再次体现了开放封闭原则（OCP）——我们只是扩展系统。

## Game Simulation

大多数棋盘游戏非常相似：游戏开始（进行某种设置），玩家轮流行动直到决定出胜者，然后宣布胜利者。无论是什么游戏——象棋、跳棋或其他游戏，我们可以将算法定义如下：

```c++
class Game
{
  void run()
  {
    start();
    while (!have_winner())
      take_turn();
    cout << "Player " << get_winner() << " wins.\n";
  }
};
```

如你所见，`run()` 方法，即运行游戏的方法，只是调用了一组其他方法。这些方法被定义为纯虚函数，并且具有 `protected` 的可见性，因此它们不会被直接调用：

```c++
protected:
  virtual void start() = 0;  
  virtual bool have_winner() = 0;
  virtual void take_turn() = 0;
  virtual int get_winner() = 0;
```

公平地说，前面提到的一些方法，特别是返回 `void` 的方法，并不一定必须是纯虚函数。例如，如果某些游戏没有显式的 `start()` 过程，将 `start()` 定义为纯虚函数会违反接口隔离原则（ISP），因为那些不需要该方法的类仍然不得不实现它。在策略模式章节中，我们特意创建了一个带有虚拟无操作（no-op）方法的策略，但对于模板方法模式，情况并不那么明确。

现在，除了前面的例子之外，我们可以有一些对所有游戏都相关的 `public` 成员：玩家的数量和当前玩家的索引：

```c++
class Game
{
public:
  explicit Game(int number_of_players)
    : number_of_players(number_of_players) {}

protected:
  int current_player{ 0 };
  int number_of_players;
}; // other members omitted
```

从这里开始，`Game` 类可以被扩展以实现象棋游戏：

```c++
class Chess : public Game
{
public:
  explicit Chess() : Game{ 2 } {}
protected:
  void start() override {}
  bool have_winner() override { return turns == max_turns; }
  void take_turn() override
  {
    turns++;
    current_player = (current_player + 1) % number_of_players;
  }
  int get_winner() override { return current_player;}
private:
  int turns{ 0 }, max_turns{ 10 };
};
```

象棋游戏涉及两名玩家，因此这一信息被传递给构造函数。然后我们继续重写所有必要的函数，实现一些非常简单的模拟逻辑，在十回合后结束游戏。以下是输出：

```
Starting a game of chess with 2 players
Turn 0 taken by player 0
Turn 1 taken by player 1
...
Turn 8 taken by player 0
Turn 9 taken by player 1
Player 0 wins.
```

就是这样，基本上就是这些内容！

## Summary

与使用组合的策略模式不同，策略模式因此分为静态和动态变体，模板方法模式使用继承，因此它只能是静态的，因为一旦对象被构造，就无法改变其继承特性。

在模板方法模式中唯一的设计决策是你是否希望模板方法所使用的方法是纯虚函数还是实际具有函数体，即使这个函数体是空的。如果你预见到某些方法对所有派生类来说并非必要，那么可以将它们实现为无操作（no-op）方法。
