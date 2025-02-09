# State

我必须承认：我的行为是由我的状态所控制的。如果我没有得到足够的睡眠，我会感到有点疲倦。如果我喝了酒，我就不会开车。所有这些都是 *state*（状态），它们决定了我的行为：我感觉如何，以及我能做什么和不能做什么。

当然，我可以从一个状态转换到另一个状态。我可以去喝杯咖啡，这将使我从困倦转变为警觉（我希望如此！）。因此，我们可以把咖啡看作是一个 *trigger*（触发器），它导致了从困倦到警觉的转变。这里，让我笨拙地为你说明一下：

```
        coffee
sleepy --------> alert
```

所以，状态设计模式是一个非常简单的概念：状态控制行为；状态可以改变；唯一有待商榷的是谁触发了状态的变化。

从根本上说，有两种方式：

- 状态是实际的类，具有行为，这些行为会在不同状态之间切换。
- 状态和转换只是枚举。我们有一个称为状态机的特殊组件来执行实际的转换。

这两种方法都是可行的，但实际上是第二种方法更为常见。我们将看一下这两种方法，但我必须承认我会简略地提及第一种，因为这并不是人们通常的做法。

## State-Driven State Transitions

我们将从最简单的例子开始：一个电灯开关。它只能处于 *on* 和 *off* 两种状态。我们将构建一个模型，其中任何状态都能够切换到其他状态：虽然这反映了“经典”的状态设计模式实现（根据 GoF 书籍），但这并不是我推荐的做法。

首先，让我们建模这个电灯开关：它所拥有的只是一个状态和一些从一个状态切换到另一个状态的手段：

```c++
class LightSwitch
{
  State *state;
public:
  LightSwitch()
  {
    state = new OffState();
  }
  void set_state(State* state)
  {
    this->state = state;
  }
};
```

这一切看起来都非常合理。我们现在可以定义状态，在这个特定的情况下，它将是一个实际的类：

```c++
struct State
{
  virtual void on(LightSwitch *ls)
  {
      cout << "Light is already on\n";
  }

  virtual void off(LightSwitch *ls)
  {
      cout << "Light is already off\n"
  }
};
```

这个实现远非直观，因此我们需要慢慢且仔细地讨论它，因为从一开始，`State` 类的任何东西看起来都没有意义。

首先，`State` 并不是抽象的！你可能会认为一个无法达到（或没有理由达到）的状态应该是抽象的。但它并不是。

其次，`State` 允许从一个状态切换到另一个状态。这……对于一个理智的人来说，是没有意义的。想象一下电灯开关：是开关改变了状态。状态本身并不期望改变自己，但这里的情况似乎正是如此。

第三，也许是最令人困惑的是，`State::on/off` 的默认行为声称我们已经处于该状态了！随着我们实现其余的例子，这一点会逐渐清晰起来。

我们现在来实现 `On` 和 `Off` 状态：

```c++
struct OnState : State
{
  OnState() { cout << "Light turned on\n"; }
  void off(LightSwitch* ls) override;
};

struct OffState : State
{
  OffState() { cout << "Light turned off\n"; }
  void on(LightSwitch* ls) override;
};
```

`OnState::off` 和 `OffState::on` 的实现允许状态本身切换到另一个状态！以下是它的样子：

```c++
void OnState::off(LightSwitch* ls)
{
  cout << "Switching light off...\n";
  ls->set_state(new OffState());
  delete this;
}   // same for OffState::on
```

所以，状态切换就发生在这里。这个实现包含了一个不常见的 `delete this` 调用，这是你在现实世界的 C++ 中很少见到的东西。`delete this` 做了一个非常危险的假设，即状态最初是在哪里分配的。这个例子可以使用智能指针重写，但是使用指针和堆分配清楚地表明了状态正在被主动销毁。如果状态有一个析构函数，它会触发，并且你可以在这里执行额外的清理工作。

当然，我们也希望开关本身能够切换状态，这看起来如下：

```c++
class LightSwitch
{
  ...
  void on() { state->on(this); }
  void off() { state->off(this); }
};
```

所以，把所有内容整合在一起，我们可以运行以下场景：

```c++
LightSwitch ls; // Light turned off
ls.on();        // Switching light on...
                // Light turned on
ls.off();       // Switching light off...
                // Light turned off
ls.off();       // Light is already off
```

我必须承认：我不喜欢这种方法，因为它并不直观。当然，状态可以被通知（通过观察者模式）我们正在进入它。但是，状态自己切换到另一个状态的想法——这是根据 GoF 书籍中的“经典”状态模式实现——似乎并不是特别令人满意。

如果我们笨拙地展示从 `OffState` 到 `OnState` 的转换，它需要被展示为：

```c++
          LightSwitch::on() -> OffState::on()
OffState -------------------------------------> OnState
```

另一方面，从 `OnState` 到 `OnState` 的转换使用了基础的 `State` 类，这个类会告诉你你已经处于该状态：

```c++
         LightSwitch::on() -> State::on()
OnState -------------------------------------> OnState
```

这里展示的例子可能显得特别人为，所以我们现在将看另一个手工设计的设置，在这个设置中，状态和转换被简化为枚举成员：

## Handmade State Machine

让我们尝试为一个典型的电话对话定义一个状态机。首先，我们将描述电话的状态：

```c++
enum class State
{
  off_hook,
  connecting,
  connected,
  on_hold,
  on_hook
};
```

我们现在也可以定义状态之间的转换，并且同样使用枚举类来表示：

```c++
enum class Trigger
{
  call_dialed,
  hung_up,
  call_connected,
  placed_on_hold,
  taken_off_hold,
  left_message,
  stop_using_phone
};
```

现在，这个状态机的确切规则，即哪些转换是可能的，需要存储在某个地方。

```c++
map<State, vector<pair<Trigger, State>>> rules;
```

这有点笨拙，但本质上，`map` 的 key 是我们要离开的状态，而值是一组 Trigger-State（触发器-状态对），表示在这个状态下可能的触发器以及使用该触发器时进入的新状态。

让我们初始化这个数据结构：

```c++
rules[State::off_hook] = {
  {Trigger::call_dialed, State::connecting},
  {Trigger::stop_using_phone, State::on_hook}
};

rules[State::connecting] = {
  {Trigger::hung_up, State::off_hook},
  {Trigger::call_connected, State::connected}
};
// more rules here
```

我们还需要一个起始状态，并且如果希望状态机在达到某个终止状态后停止执行，我们也可以添加一个退出（终端）状态：

```c++
State currentState{ State::off_hook },
exitState{ State::on_hook };
```

完成了这些设置后，我们不一定非得构建一个单独的组件来实际运行（我们使用术语 orchestrating（协调））状态机。例如，如果我们想要构建一个电话的交互模型，我们可以这样做：

```c++
while(true)
{
  cout << "The phone is currently " << currentState << endl;
select_trigger:
  cout << "Select a trigger:" << "\n";

  int i = 0;
  for (auto item : rules[currentState])
  {
      cout << i++ << ". " << item.first << "\n";
  }

  int input;
  cin >> input;
  if (input < 0 || (input+1) > rules[currentState].size())
  {
      cout << "Incorrect option. Please try again." << "\n";
      goto select_trigger;
  }

  currentState = rules[currentState][input].second;
  if (currentState == exitState) break;
}
```

首先：是的，我确实使用了 `goto`，这是一个适当使用它的良好示例。至于算法本身，这是相当明显的：我们让用户选择当前状态下可用的一个触发器（背后已经为 `State` 和 `Trigger` 实现了操作符 `<<`），并且只要触发器有效，我们就通过之前创建的 `rules` 映射进行状态转换。

如果我们到达的状态是退出状态，我们就直接跳出循环。以下是一个与程序交互的示例：

```c++
The phone is currently off the hook
Select a trigger:
1. call dialed
2. putting phone on hook
0
The phone is currently connecting
Select a trigger:
1. hung up
2. call connected
1
The phone is currently connected
Select a trigger:
1. left message
2. hung up
3. placed on hold
2
The phone is currently on hold
Select a trigger:
1. taken off hold
2. hung up
1
The phone is currently off the hook
Select a trigger:
1. call dialed
2. putting phone on hook
1
We are done using the phone
```

这个手工实现的状态机的主要优点是非常容易理解：状态和转换是普通的枚举，转换集合是在一个简单的 `std::map` 中定义的，而起始和结束状态是简单的变量。

## State Machines with Boost.MSM

在现实世界中，状态机更为复杂。有时，你希望在达到某个状态时触发某些动作。其他时候，你希望转换是有条件的，也就是说，你希望只有在满足某些条件的情况下才发生状态转换。

当使用 Boost.MSM（Meta State Machine），这是 Boost 库的一部分，你的状态机是一个通过 CRTP（Curiously Recurring Template Pattern，奇特递归模板模式）从 `state_machine_def` 继承的类：

```c++
struct PhoneStateMachine : state_machine_def<PhoneStateMachine>
{
  bool angry{ false };
```

我添加了一个 `bool` 来表示呼叫者是否生气（例如，因为被置于等待状态）；我们稍后会用到它。现在，每个状态也可以存在于状态机中，并且预期要从 `state` 类继承：

```c++
struct OffHook : state<> {};
struct Connecting : state<>
{
  template <class Event, class FSM>
  void on_entry(Event const& evt, FSM&)
  {
      cout << "We are connecting..." << endl;
  }
  // also on_exit
};
// other states omitted
```

如你所见，状态还可以定义在进入或退出特定状态时发生的行为。

你也可以定义在转换过程中（而不是在达到某个状态时）执行的行为：这些也是类，但它们不需要继承自任何类；相反，它们需要提供具有特定签名的 `operator()`：

```c++
struct PhoneBeingDestroyed
{
  template <class EVT, class FSM, class SourceState, class TargetState>
  void operator()(EVT const&, FSM&, SourceState&, TargetState&)
  {
      cout << "Phone breaks into a million pieces" << endl;
  }
};
```

如你可能已经猜到的，参数为你提供了对状态机以及你正在离开和进入的状态的引用。

最后，我们有 * guard conditions*：这些条件决定了我们是否可以首先使用某个转换。现在，我们的布尔变量 `angry` 不是以 MSM 可用的形式存在的，所以我们需要将其包装：

```c++
struct CanDestroyPhone
{
  template <class EVT, class FSM, class SourceState, class TargetState>
  bool operator()(EVT const&, FSM& fsm, SourceState&, class TargetState)
  {
      return fsm.angry;
  }
}
```

前面的内容创建了一个名为 `CanDestroyPhone` 的保护条件，我们可以在定义状态机时稍后使用。

在定义状态机规则时，Boost.MSM 使用了 MPL（元编程库）。具体来说，转换表被定义为一个 `mpl::vector`，每一行依次包含：

- 源状态
- 转换事件
- 目标状态
- 可选的执行动作
- 可选的保护条件

因此，综合所有这些内容，我们可以如下定义一些电话呼叫的规则：

```c++
struct transition_table : mpl::vector<
  Row<OffHook, CallDialed, Connecting>,
  Row<Connecting, CallConnected, Connected>,
  Row<Connected, PlacedOnHold, OnHold>,
  Row<OnHold, PhoneThrownIntoWall, PhoneDestroyed, PhoneBeingDestroyed, CanDestroyPhone>
> {};
```

在上述内容中，与状态不同的是，像 `CallDialed` 这样的转换是可以在状态机类外部定义的类。它们不需要继承自任何基类，并且可以很简单甚至为空，但它们必须是类型。

我们 ` transition_table` 的最后一行是最有趣的：它指定了只有在满足 `CanDestroyPhone` 保护条件的情况下，才可以尝试销毁电话，并且当电话实际上被销毁时，应该执行 `PhoneBeingDestroyed` 动作。

现在，我们还可以添加一些其他的东西。首先，我们添加起始条件：由于我们使用的是 Boost.MSM，起始条件是一个 `typedef`，而不是一个变量：

```c++
typedef OffHook initial_state;
```

最后，我们可以定义一个动作，以备没有可能的转换时发生。这种情况是可能发生！例如，在你砸坏了电话之后，你就不能再使用它了，对吧？

```c++
template <class FSM, class Event>
void no_transition(Event const& e, FSM&, int state)
{
  cout << "No transition from state " << state_names[state] << " on event " << typeid(e).name() << endl;
}
```

Boost MSM 将状态机分为前端（我们刚才编写的部分）和后端（运行它的部分）。使用后端 API，我们可以根据前面的状态机定义构造状态机：

```c++
msm::back::state_machine<PhoneStateMachine> phone;
```

现在，假设存在一个 `info()` 函数，它只是打印我们当前所处的状态，我们可以尝试协调以下场景：

```c++
info(); // The phone is currently off hook
phone.process_event(CallDialed{}); // We are connecting...
info(); // The phone is currently connecting
phone.process_event(CallConnected{});
info(); // The phone is currently connected
phone.process_event(PlacedOnHold{});
info(); // The phone is currently on hold

phone.process_event(PhoneThrownIntoWall{});
// Phone breaks into a million pieces

info(); // The phone is currently destroyed

phone.process_event(CallDialed{});
// No transition from state destroyed on event struct CallDialed
```

这就是如何定义一个更为复杂、工业级强度的状态机。

## 总结

首先，值得强调的是，Boost.MSM 是 Boost 中两种状态机实现之一，另一种是 Boost.Statechart。我相当确定还有很多其他的状态机实现。

其次，状态机的功能远不止这些。例如，许多库支持分层状态机的概念：比如，状态 `Sick` 可以包含许多不同的子状态，如 `Flu` 或 `Chickenpox`。如果你处于 `Flu` 状态，那么也意味着你处于 `Sick` 状态。

最后，再次强调现代状态机与最初形式的状态设计模式之间的差异是有价值的。存在重复的 API（例如，`LightSwitch::on/off` 与 `State::on/off`）以及自我删除的存在，在我看来这些都是代码异味。别误会我的意思——这种方法确实可行，但它不够直观且繁琐。
