# Mediator

我们编写的大比例代码中，不同的组件（类）通过直接的引用或指针相互通信。然而，在某些情况下，您不希望对象必然意识到彼此的存在。或者，也许您确实希望它们彼此知晓，但仍然不希望通过指针或引用进行通信，因为这些指针或引用可能会失效，您也不想解引用一个 `nullptr`，对吧？

因此，中介者（Mediator）是一种促进组件之间通信的机制。自然地，中介者本身需要对每个参与的组件可访问，这意味着它应该要么是一个全局静态变量，或者替代地，只是一个被注入到每个组件中的引用。

## Chat Room

典型的互联网聊天室是中介者（Mediator）设计模式的经典例子，所以在我们转向更复杂的内容之前，让我们先实现这个。

最简单的聊天室参与者实现可以简单到如下：

```c++
struct Person
{
  string name;
  ChatRoom* room = nullptr;
  vector<string> chat_log;

  Person(const string& name);

  void receive(const string& origin, const string& message);
  void say(const string& message) const;
  void pm(const string& who, const string& message) const;
};
```

所以我们有一个具有名字（用户ID）、聊天记录和指向实际 `ChatRoom` 的指针的人。我们有一个构造函数以及三个成员函数：

- `receive()` 允许我们接收消息。通常这个函数会将消息显示在用户的屏幕上，并将其添加到聊天记录中。
- `say()` 允许该人向房间中的每个人广播消息。
- `pm()` 是私信功能。您需要指定消息的接收者的名字。

`say()` 和 `pm()` 只是将操作转发给聊天室。说到这个，让我们实际上实现 `ChatRoom`——它并不是特别复杂：

```c++
struct ChatRoom
{
  vector<Person*> people; // assume append-only

  void join(Person* p);
  void broadcast(const string& origin, const string& message);
  void message(const string& origin, const string& who,
    const string& message);
};
```

是否使用指针、引用或 `shared_ptr` 来实际存储聊天室参与者的列表最终取决于您：唯一的限制是 `std::vector` 不能存储引用。因此，我决定在这里使用指针。`ChatRoom` 的 API 非常简单：

- `join()` 让一个人加入房间。我们不会实现 `leave()`，而是把这个想法留到本章的后续示例中。
- `broadcast()` 向所有人发送消息……嗯，不是所有人：我们不需要把消息再发回给发送者本人。
- `message()` 发送私信。

`join()` 的实现如下：

```c++
void ChatRoom::join(Person* p)
{
  string join_msg = p->name + " joins the chat";
  broadcast("room", join_msg);
  p->room = this;
  people.push_back(p);
}
```

就像经典的 IRC 聊天室一样，我们向房间中的每个人广播有人加入的消息。在这种情况下，消息的来源被指定为 "room" 而不是加入的人。然后我们设置该人的 `room` 指针，并将他们添加到房间中的人的列表中。

现在，让我们来看看 `broadcast()`：这是消息发送给每个房间参与者的地方。请记住，每个参与者都有自己的 `Person::receive()` 函数来处理消息，因此实现相对简单：

```c++
void ChatRoom::broadcast(const string& origin, const string& message)
{
  for (auto p : people)
    if (p->name != origin)
      p->receive(origin, message);
}
```

是否希望防止广播消息被发送给自己是一个有争议的问题，但在这里我积极避免这种情况。不过，其他所有人都会收到消息。

最后，以下是使用 `message()` 实现的私信功能：

```c++
void ChatRoom::message(const string& origin,
  const string& who, const string& message)
{
  auto target = find_if(begin(people), end(people),
    [&](const Person* p) { return p->name == who; });
  if (target != end(people))
  {
    (*target)->receive(origin, message);
  }
}
```

这会在 `people` 列表中查找接收者，并且如果找到接收者（因为有可能他们已经离开了房间），则将消息分发给那个人。

回到 `Person` 的 `say()` 和 `pm()` 实现，它们如下所示：

```c++
void Person::say(const string& message) const
{
  room->broadcast(name, message);
}

 void Person::pm(const string& who, const string& message) const
{
  room->message(name, who, message);
}
```

至于 `receive()`，这是一个很好的地方来实际显示消息在屏幕上，并将其添加到聊天记录中。

```c++
void Person::receive(const string& origin, const string& message)
{
  string s{ origin + ": \"" + message + "\"" };
  cout << "[" << name << "'s chat session] " << s << "\n";
  chat_log.emplace_back(s);
}
```

我们在这里多走了一步，不仅显示消息来自谁，还显示我们当前处于哪个用户的聊天会话中——这将有助于诊断谁在何时说了什么。

以下是我们将要运行的场景：

```c++
ChatRoom room;

Person john{ "john" };
Person jane{ "jane" };
room.join(&john);
room.join(&jane);
john.say("hi room");
jane.say("oh, hey john");

Person simon("simon");
room.join(&simon);
simon.say("hi everyone!");

jane.pm("simon", "glad you could join us, simon");
```

以下是输出结果：

```
[john's chat session] room: "jane joins the chat"
[jane's chat session] john: "hi room"
[john's chat session] jane: "oh, hey john"
[john's chat session] room: "simon joins the chat"
[jane's chat session] room: "simon joins the chat"
[john's chat session] simon: "hi everyone!"
[jane's chat session] simon: "hi everyone!"
[simon's chat session] jane: "glad you could join us, simon"
```

## Mediator with Events

在聊天室的例子中，我们遇到了一个一致的主题：每当有人发布消息时，参与者需要收到通知。这看起来像是观察者（Observer）模式的完美场景，该模式在第20章中有讨论：中介者拥有一个所有参与者共享的事件；参与者可以订阅该事件以接收通知，也可以触发事件，从而引发通知。

事件并不是 C++ 内置的功能（不像例如 C#），因此我们将为这个演示使用一个库解决方案。Boost.Signals2 提供了所需的功能，尽管术语略有不同：我们通常谈论的是 `signals`（生成通知的对象）和 `slots`（处理通知的函数）。

为了避免再次重做聊天室，让我们来看一个更简单的例子：想象一场足球比赛（soccer），有球员和足球教练。当教练看到他们的球队进球时，他们自然想要恭喜进球的球员。当然，他们需要一些关于事件的信息，比如谁进了球以及他们迄今为止进了多少球。

我们可以引入一个用于任何类型事件数据的基础类：

```c++
struct EventData
{
  virtual ~EventData() = default;
  virtual void print() const = 0;
};
```

我添加了 `print()` 函数，以便每个事件都可以打印到命令行，并且还添加了一个虚析构函数以使 ReSharper 不再对此发出警告。现在，我们可以从这个类派生，以便存储一些与进球相关的数据：

```c++
struct PlayerScoredData : EventData
{
  string player_name;
  int goals_scored_so_far;

  PlayerScoredData(const string& player_name, const int goals_scored_so_far)
    : player_name(player_name),
      goals_scored_so_far(goals_scored_so_far) {}

  void print() const override
  {
    cout << player_name << " has scored! (their "
      << goals_scored_so_far << " goal)" << "\n";
  }
};
```

我们再次构建一个中介者，但它将没有任何行为！认真地说，有了事件驱动的基础设施，这些行为不再需要：

```c++
struct Game
{
  signal<void(EventData*)> events; // observer
};
```

事实上，您可以用一个全局 `signal` 来解决问题，完全不需要创建 `Game` 类，但我们在这里使用最小惊讶原则（principle of least surprise），如果将 `Game&` 注入到组件中，我们知道那里有一个明确的依赖关系。

无论如何，我们现在可以构建 `Player` 类。一个球员有一个名字、在比赛中进球的数量，当然还有一个对中介者 `Game` 的引用：

```c++
struct Player
{
  string name;
  int goals_scored = 0;
  Game& game;

  Player(const string& name, Game& game)
    : name(name), game(game) {}

  void score()
  {
    goals_scored++;
    PlayerScoredData ps{name, goals_scored};
    game.events(&ps);
  }
};
```

`Player::score()` 是这里有趣的功能：它使用 `events` 信号来创建一个 `PlayerScoredData` 并将其发布给所有订阅者查看。谁会收到这个事件？当然是教练（Coach）：

```c++
struct Coach
{
  Game& game;
  explicit Coach(Game& game) : game(game)
  {
    // celebrate if player has scored <3 goals
    game.events.connect([](EventData* e)
    {
      PlayerScoredData* ps = dynamic_cast<PlayerScoredData*>(e);
      if (ps && ps->goals_scored_so_far < 3)
      {
        cout << "coach says: well done, " << ps->player_name << "\n";
      }
    });
  }
};
```

`Coach` 类的实现非常简单；我们的教练甚至没有名字。但我们确实给他提供了一个构造函数，在其中创建了对 `game.events` 的订阅，这样每当有事件发生时，教练可以通过提供的 lambda（slot）来处理事件数据。

请注意，lambda 的参数类型是 `EventData*`——我们不知道是球员进球了还是被罚下了，所以我们需要使用 `dynamic_cast`（或类似的机制）来确定我们得到了正确的类型。

有趣的是，所有的魔法都在设置阶段发生：不需要为特定信号显式注册槽函数。客户端可以自由地使用构造函数创建对象，然后当球员进球时，通知就会自动发送：

```c++
Game game;
Player player{ "Sam", game };
Coach coach{ game };

player.score();
player.score();
player.score(); // ignored by coach
```

这产生了以下输出：

```
coach says: well done, Sam
coach says: well done, Sam
```

输出只有两行长，因为到第三个进球时，教练已经不再印象深刻了。

## Summary

中介者（Mediator）设计模式本质上建议引入一个中间组件，系统中的每个对象都有对该组件的引用，并可以使用它来进行相互通信。代替直接的内存地址，通信可以通过标识符（用户名、唯一ID等）进行。

中介者的最简单实现是一个成员列表和一个遍历列表并执行其预定功能的函数——无论是对列表中的每个元素还是选择性地对某些元素操作。

更复杂的中介者实现可以使用事件机制，允许参与者订阅（和取消订阅）系统中发生的事情。这样，从一个组件发送到另一个组件的消息可以被当作事件来处理。在这种设置下，如果参与者不再对某些事件感兴趣或即将离开系统，他们也很容易取消订阅这些事件。
