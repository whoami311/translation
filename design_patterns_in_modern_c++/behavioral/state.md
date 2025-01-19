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



```c++
class LightSwitch
{
  ...
  void on() { state->on(this); }
  void off() { state->off(this); }
};
```

```c++
LightSwitch ls; // Light turned off
ls.on();        // Switching light on...
                // Light turned on
ls.off();       // Switching light off...
                // Light turned off
ls.off();       // Light is already off
```

```c++
          LightSwitch::on() -> OffState::on()
OffState -------------------------------------> OnState
```

```c++
         LightSwitch::on() -> State::on()
OnState -------------------------------------> OnState
```

## Handmade State Machine

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

```c++
map<State, vector<pair<Trigger, State>>> rules;
```

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

```c++
State currentState{ State::off_hook },
exitState{ State::on_hook };
```

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

## State Machines with Boost.MSM

```c++
struct PhoneStateMachine : state_machine_def<PhoneStateMachine>
{
    bool angry{ false };
```

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

```c++
struct CanDestroyPhone
{
    template <class EVT, class FSM, class SourceState, class TargetState>
    bool operator()(EVT const&, FSM& fsm, SourceState&, class TargetState)
    {
        return fsm.angry;
    }
};
```

```c++
struct transition_table : mpl::vector<
    Row<OffHook, CallDialed, Connecting>,
    Row<Connecting, CallConnected, Connected>,
    Row<Connected, PlacedOnHold, OnHold>,
    Row<OnHold, PhoneThrownIntoWall, PhoneDestroyed, PhoneBeingDestroyed, CanDestroyPhone>
> {};
```

```c++
typedef OffHook initial_state;
```

```c++
template <class FSM, class Event>
void no_transition(Event const& e, FSM&, int state)
{
    cout << "No transition from state " << state_names[state] << " on event " << typeid(e).name() << endl;
}
```

```c++
msm::back::state_machine<PhoneStateMachine> phone;
```

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

## Summary
