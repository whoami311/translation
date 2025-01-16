# State

```
        coffee
sleepy --------> alert
```

## State-Driven State Transitions

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
