# eUML

## eUML

重要提示：eUML 需要一个支持 Boost.Typeof 的编译器。完整的 eUML 具有实验性状态（但如果仅转换表使用 eUML 编写，则不是），因为当状态机变得过大时（通常是当您编写巨大的动作时），某些编译器将开始崩溃。

之前的前端虽然易于编写，但仍会引入一定量的噪音，主要是 MPL 类型，因此能够直接在转换表中编写类似于 C++ 的代码（使用 C++ 动作语言）将会很好，就像 UML 设计师喜欢在他们的状态机图上所做的那样。如果是函数式编程，那就更好了。这就是 eUML 的用途。

eUML 是基于 Boost.Proto 和 Boost.Typeof 的编译时领域特定嵌入式语言。它提供了允许直接在转换表或状态定义中的入口/出口处定义动作/保护条件的语法。这些语法包括动作、保护条件、标志、属性、延迟事件和初始状态。

它还依赖于 Boost.Typeof 作为新 C++0x 特性 decltype 的包装器，以提供对所有语法的编译时评估。不幸的是，并非所有底层的 Boost 库都启用了 Typeof 支持，因此目前您需要一个支持 Typeof 的编译器（例如 VC9-10、g++ >= 4.3）。

示例将在接下来的段落中提供。您需要包含 eUML 的基本功能：

```c++
#include <msm/front/euml/euml.hpp>
```

为了添加 STL 支持（可能会导致编译时间更长），请包含：

```c++
#include <msm/front/euml/stl.hpp>
```

eUML 定义在命名空间 `msm::front::euml` 中。

## Transition table

可以使用 eUML 定义一个转换，如下所示：

```c++
source + event [guard] / action == target
```

或者如下：

```c++
target == source + event [guard] / action
```

第一个版本看起来像图表中绘制的转换，第二个版本对 C++ 开发人员来说则显得自然。

使用函子前端编写的简单转换表现在可以写成：

```c++
BOOST_MSM_EUML_TRANSITION_TABLE(( 
Stopped + play [some_guard] / (some_action , start_playback)  == Playing ,
Stopped + open_close/ open_drawer                             == Open    ,
Stopped + stop                                                == Stopped ,
Open    + open_close / close_drawer                           == Empty   ,
Empty   + open_close / open_drawer                            == Open    ,
Empty   + cd_detected [good_disk_format] / store_cd_info      == Stopped
),transition_table)    
```

或者，使用替代表示法，可以写为：

```c++
BOOST_MSM_EUML_TRANSITION_TABLE(( 
Playing  == Stopped + play [some_guard] / (some_action , start_playback) ,
Open     == Stopped + open_close/ open_drawer                            ,
Stopped  == Stopped + stop                                               ,
Empty    == Open    + open_close / close_drawer                          ,
Open     == Empty   + open_close / open_drawer                           ,
Stopped  == Empty   + cd_detected [good_disk_format] / store_cd_info
),transition_table)    
```

转换表现在看起来像是一系列（可读的）规则，几乎没有噪音。

UML 在“[ ]”之间定义保护条件，在“/”之后定义动作，因此所选择的语法对 UML 设计师来说已经更易读。UML 还允许设计师通过逗号分隔顺序定义多个动作（我们之前的 `ActionSequence_`）。第一个转换正是这样做的：两个动作用逗号分隔并用括号括起来以符合 C++ 操作符优先级。

如果您担心这会影响运行时性能，请不要担心，eUML 基于 `typeof`（或 `decltype`），它仅评估传递给 `BOOST_MSM_EUML_TRANSITION_TABLE` 的参数，并且不会产生任何运行时开销。实际上，eUML 只是“标准”MSM 元编程之上的一个元编程层，而这一层生成了之前介绍的函子前端。

UML 还允许设计师定义更复杂的保护条件，例如 `[good_disk_format && (some_condition || some_other_condition)]`。使用我们之前定义的函子这是可能的，但需要使用复杂的模板语法。现在这种语法可以完全按照书写的方式实现，这意味着完全没有语法噪音。

## A simple example: rewriting only our transition table

作为对 eUML 的介绍，我们将使用 eUML 重写教程中的转换表。这将需要两到三个更改，具体取决于编译器：

- 事件必须继承自 `msm::front::euml::euml_event<event_name>`
- 状态必须继承自 `msm::front::euml::euml_state<state_name>`
- 使用 VC 时，状态必须在前端之前声明

我们现在可以像刚刚展示的那样编写转换表，使用 `BOOST_MSM_EUML_DECLARE_TRANSITION_TABLE` 而不是 `BOOST_MSM_EUML_TRANSITION_TABLE`。实现非常直接。唯一需要添加的是为每个状态声明一个变量或在转换表中添加括号（默认构造函数调用）。

复合实现也很自然：

```c++
// front-end like always
struct sub_front_end : public boost::msm::front::state_machine_def<sub_front_end>
{
...
};
// back-end like always
typedef boost::msm::back::state_machine<sub_front_end> sub_back_end;

sub_back_end const sub; // sub can be used in a transition table.
```

不幸的是，使用 VC 时存在一个 bug，它偶尔会出现并导致堆栈溢出。如果收到程序在所有路径上都是递归的警告，请回退到标准 eUML 或其他前端，因为微软似乎不打算修复这个问题。

我们现在有了一个新的、更易读的转换表，并且对示例的更改很少。eUML 可以做更多的事情，请继续阅读指南。

## Defining events, actions and states with entry/exit actions

### Events

事件必须启用 proto。为此，它们必须继承自一个 proto 终端（`euml_event<event_name>`）。eUML 还提供了一个宏来简化这一过程：

```c++
BOOST_MSM_EUML_EVENT(play)
```

这声明了一个事件类型和该类型的实例 `play`，它现在可以在状态或转换行为中使用。

还有一个名为 `BOOST_MSM_EUML_EVENT_WITH_ATTRIBUTES` 的宏，它将事件将包含的属性作为第二个参数，使用属性语法。

注意：由于我们现在将事件定义为实例而不是单纯的类型，我们是否仍然可以通过动态创建一个事件来处理它，例如：`fsm.process_event(play());` 或者我们必须写成：`fsm.process_event(play);`

答案是你可以使用这两种方式。第二种方式更简单，但与其他前端不同的是，第二种方式使用一个定义的 `operator()`，它会在运行时动态创建一个事件。

### Actions

动作（返回 `void`）和保护条件（返回 `bool`）的定义方式与之前的函子类似，区别在于它们也必须启用 proto。这可以通过继承 `euml_action<functor_name>` 来实现。eUML 还提供了一个宏：

```c++
BOOST_MSM_EUML_ACTION(some_condition)
{
    template <class Fsm,class Evt,class SourceState,class TargetState>
    bool operator()(Evt const& ,Fsm& ,SourceState&,TargetState& ) 
    { return true; }
};
```

与事件一样，这个宏声明了一个函子类型和一个实例，以便在转换或状态行为中使用。

可以使用与转换表相同的动作语法来定义状态进入和退出行为。因此，(action1, action2) 是一个有效的进入或退出行为，它会依次执行这两个动作。

状态函子的签名略有不同，因为没有源状态和目标状态，只有当前状态（进入/退出动作是与转换无关的），例如：

```c++
BOOST_MSM_EUML_ACTION(Empty_Entry)
{
    template <class Evt,class Fsm,class State>
    void operator()(Evt const& ,Fsm& ,State& ) { ... }                           
    }; 
```

也可以重用函子前端中的函子。不过，语法稍微不够简洁，因为我们需要假装动态创建一个函子以用于 typeof。例如：

```c++
struct start_playback 
{
        template <class Fsm,class Evt,class SourceState,class TargetState>
        void operator()(Evt const& ,Fsm&,SourceState& ,TargetState& )
        {
         ...            
        }
};
BOOST_MSM_EUML_TRANSITION_TABLE((
Playing   == Stopped  + play        / start_playback() ,
...
),transition_table)
```

### States

还有一个用于状态的宏。这个宏有两个参数，第一个是定义状态的表达式，第二个是状态（实例）的名称：

```c++
BOOST_MSM_EUML_STATE((),Paused)
```

这定义了一个没有进入或退出动作的简单状态。你可以在表达式参数中使用动作语法提供状态行为（进入和退出），就像在转换表中一样：

```c++
BOOST_MSM_EUML_STATE(((Empty_Entry,Dummy_Entry)/*2 entryactions*/,
                       Empty_Exit/*1 exit action*/ ),
                     Empty)
```

这意味着 `Empty` 被定义为一个状态，其进入动作由两个子动作 `Empty_Entry` 和 `Dummy_Entry`（用括号括起来）组成，并且有一个退出动作 `Empty_Exit`。

表达式语法有几种可能性：

- `()`: 没有进入或退出动作的状态。
- `(Expr1)`: 有进入动作但没有退出动作的状态。
- `(Expr1, Expr2)`: 有进入和退出动作的状态。
- `(Expr1, Expr2, Attributes)`: 有进入和退出动作的状态，定义了一些属性（详见下文）。
- `(Expr1, Expr2, Attributes, Configure)`: 有进入和退出动作的状态，定义了一些属性（详见下文）和标志（标准 MSM 标志）或延迟事件（标准 MSM 延迟事件）。
- `(Expr1, Expr2, Attributes, Configure, Base)`: 有进入和退出动作的状态，定义了一些属性（详见下文）、标志和延迟事件（普通的 MSM 延迟事件）以及非默认基状态（如标准 MSM 中所定义）。

还定义了 `no_action`，它除了作为一个占位符外不做任何事情（例如，当我们没有进入动作但有退出动作时，作为进入动作）。`Expr1` 和 `Expr2` 是一系列动作，遵循与转换表中相同的动作语法（跟随“/”符号）。

`BOOST_MSM_EUML_STATE` 宏允许你定义大多数常见的状态，但有时你需要更多功能，例如在你的状态中提供一些特殊行为。在这种情况下，你需要手动完成宏的工作，这并不复杂。状态需要继承自 `msm::front::state<>`，像任何状态一样，并且从 `euml_state<state_name>` 继承以启用 proto。然后你需要声明一个实例以在转换表中使用。例如：

```c++
struct Empty_impl : public msm::front::state<> , public euml_state<Empty_impl> 
{
   void activate_empty() {std::cout << "switching to Empty " << std::endl;}
   template <class Event,class Fsm>
   void on_entry(Event const& evt,Fsm&fsm){...}
   template <class Event,class Fsm>
   void on_exit(Event const& evt,Fsm&fsm){...}
};
//instance for use in the transition table
Empty_impl const Empty;
```

请注意，我们还定义了一个名为 `activate_empty` 的方法。我们希望在某个行为中调用它。这可以通过使用 `BOOST_MSM_EUML_METHOD` 宏来实现。

```c++
BOOST_MSM_EUML_METHOD(ActivateEmpty_,activate_empty,activate_empty_,void,void)
```

第一个参数是底层函子的名称，你可以将其与函子前端一起使用；第二个参数是状态方法的名称；第三个参数是 eUML 生成的函数；第四个和第五个参数是在转换或状态行为中使用时的返回值。你现在可以在转换中使用这个方法：

```c++
Empty == Open + open_close / (close_drawer,activate_empty_(target_))
```

## Wrapping up a simple state machine and first complete examples

你可以重用标准前端中的状态机定义方法，只需将转换表替换为这个新的转换表即可。你也可以使用 eUML 来“即时”定义一个状态机（例如，如果你需要为该状态机提供一个作为函子的 `on_entry`/`on_exit`）。为此，还有一个宏 `BOOST_MSM_EUML_DECLARE_STATE_MACHINE`，它有两个参数：描述状态机的表达式和状态机名称。该表达式最多可以有 8 个参数：

- `(Stt, Init)`：最简单的状态机，仅定义了转换表和初始状态。
- `(Stt, Init, Expr1)`：定义了转换表、初始状态和进入动作的状态机。
- `(Stt, Init, Expr1, Expr2)`：定义了转换表、初始状态、进入动作和退出动作的状态机。
- `(Stt, Init, Expr1, Expr2, Attributes)`：定义了转换表、初始状态、进入动作和退出动作的状态机，并添加了一些属性（详见下文）。
- `(Stt, Init, Expr1, Expr2, Attributes, Configure)`：定义了转换表、初始状态、进入动作和退出动作的状态机，并添加了一些属性（详见下文）、标志、延迟事件以及配置能力（无消息队列/无异常捕获）。
- `(Stt, Init, Expr1, Expr2, Attributes, Flags, Deferred, Base)`：定义了转换表、初始状态、进入动作和退出动作的状态机，并添加了一些属性（详见下文）、标志、延迟事件、配置能力（无消息队列/无异常捕获），还定义了一个非默认基状态（详见后端描述）。

例如，一个最小的状态机可以定义为：

```c++
BOOST_MSM_EUML_TRANSITION_TABLE(( 
),transition_table)
```

```c++
BOOST_MSM_EUML_DECLARE_STATE_MACHINE((transition_table,init_ << Empty ),
                                     player_)
```

请查看使用 eUML 第一种语法和第二种语法编写的播放器教程。`BOOST_MSM_EUML_DECLARE_ATTRIBUTE` 宏（我们稍后会详细讨论）使用属性语法声明赋予 eUML 类型（状态或事件）的属性。

### Defining a submachine

使用其他前端定义子状态机（参见教程）只需在另一个状态机的转换表中使用一个作为状态机的状态。使用 eUML 时也是如此。你只需要定义第二个状态机，并在包含状态机的转换表中引用它。

与状态或事件定义宏不同，`BOOST_MSM_EUML_DECLARE_STATE_MACHINE` 定义了一个类型，而不是实例，因为后端需要的是类型。这意味着你需要自己声明一个实例，以便在另一个状态机中引用你的子状态机，例如：

```c++
BOOST_MSM_EUML_DECLARE_STATE_MACHINE(...,Playing_)
typedef msm::back::state_machine<Playing_> Playing_type;
Playing_type const Playing;
```

我们现在可以在包含状态机的转换表中使用这个实例：

```c++
Paused == Playing + pause / pause_playback
```

### Attributes / Function call

我们现在希望让我们的语法更有用。很多时候，只需要非常简单的动作方法，例如 `++Counter` 或 `Counter > 5`，其中 `Counter` 通常定义为包含状态机的类的某个属性。为这样的简单动作编写一个函子似乎是一种浪费。此外，MSM 中的状态也是类，因此它们可以具有属性，我们也希望能够为它们提供属性。

如果你回顾我们使用第一种和第二种语法的示例，你会发现一个 `BOOST_MSM_EUML_DECLARE_ATTRIBUTE` 宏和一个 `BOOST_MSM_EUML_ATTRIBUTES` 宏。第一个宏用于声明可能的属性：

```c++
BOOST_MSM_EUML_DECLARE_ATTRIBUTE(std::string,cd_name)
BOOST_MSM_EUML_DECLARE_ATTRIBUTE(DiskTypeEnum,cd_type)
```

这声明了两个属性：类型为 `std::string` 的 `cd_name` 和类型为 `DiskTypeEnum` 的 `cd_type`。这些属性并不属于任何特定的事件或状态，我们只是声明了一个名称和类型。现在，我们可以使用第二个宏将属性添加到我们的 `cd_detected` 事件中：

```c++
BOOST_MSM_EUML_ATTRIBUTES((attributes_ << cd_name << cd_type ), 
                          cd_detected_attributes)
```

这声明了一个属性列表，目前还没有链接到任何特定的对象。它可以附加到一个状态或事件上。例如，如果我们希望 `cd_detected` 事件拥有这些定义的属性，我们可以这样写：

```c++
BOOST_MSM_EUML_EVENT_WITH_ATTRIBUTES(cd_detected,cd_detected_attributes)
```

对于状态，我们使用 `BOOST_MSM_EUML_STATE` 宏，它有一个表达式形式，可以在其中提供属性。例如：

```c++
BOOST_MSM_EUML_STATE((no_action /*entry*/,no_action/*exit*/,
                      attributes_ << cd_detected_attributes),
                     some_state)
```

很好，我们现在有了一种为类添加属性的方法，虽然这种方法可能比我们以往用的更复杂，但这有什么意义呢？关键在于，我们现在可以在转换表中直接在编译时引用这些属性。例如，在示例中，你会发现这样的转换：

```c++
Stopped==Empty+cd_detected[good_disk_format&&(event_(cd_type)==Int_<DISK_CD>())]
```

将 `event_(cd_type)` 理解为 `event_->cd_type`，其中 `event_` 是一个通用的事件类型，无论具体的事件是什么（在这个特定的例子中，正如转换所示，它是 `cd_detected`）。

这个功能的主要优点是，你不需要定义一个新的函子，并且不需要查看函子内部来了解它做什么，所有信息都一目了然。

MSM 提供了更多用于状态机类型的通用对象：

- `event_`：在任何动作中使用，触发转换的事件
- `state_`：在进入和退出动作中使用，进入或退出的状态
- `source_`：在转换动作中使用，源状态
- `target_`：在转换动作中使用，目标状态
- `fsm_`：在任何动作中使用，处理转换的（最底层的）状态机
- `Int_<int value>`：表示一个 `int` 的函子
- `Char_<value>`：表示一个 `char` 的函子
- `Size_t_<value>`：表示一个 `size_t` 的函子
- `String_<mpl::string>`（Boost >= 1.40）：表示一个字符串的函子

这些辅助工具可以以两种不同的方式使用：

- `helper(attribute_name)` 返回名称为 `attribute_name` 的属性
- `helper` 返回状态/事件类型本身

第二种形式在你想为状态提供自己的方法并且还希望在转换表中使用这些方法时非常有用。在上面的教程中，我们为 `Empty` 提供了一个 `activate_empty` 方法。我们希望创建一个 eUML 函子并从转换表中调用它。这可以通过 `MSM_EUML_METHOD` / `MSM_EUML_FUNCTION` 宏来实现。第一个宏创建一个指向方法的函子，第二个宏创建一个指向自由函数的函子。在教程中，我们编写：

```c++
MSM_EUML_METHOD(ActivateEmpty_,activate_empty,activate_empty_,void,void)
```

第一个参数是函子名称，用于与函子前端一起使用。第二个是要调用的方法的名称。第三个是用于 eUML 的函数名称，第四个是在转换动作上下文中使用的函数返回类型，第五个是在状态进入/退出动作上下文中使用的结果类型（通常第四和第五个参数是相同的）。我们现在有一个新的 eUML 函数来调用“某个对象”的方法，而这个“某个对象”就是之前展示的五个通用辅助工具之一。我们现在可以在转换中使用这个函数，例如：

```c++
Empty == Open + open_close / (close_drawer,activate_empty_(target_))
```

动作现在被定义为两个动作的序列：`close_drawer` 和 `activate_empty`，这些动作将在目标本身上调用。由于目标是 `Empty`（左侧定义的状态），这实际上会调用 `Empty::activate_empty()`。这个方法也可以有一个（或多个）参数，例如事件，那么我们可以调用 `activate_empty_(target_, event_)`。

更多示例可以在可怕的编译器压力测试、计时器示例或使用 eUML 的 iPodSearch 示例中找到（针对 `String_` 等更多内容）。

## Orthogonal regions, flags, event deferring

定义正交区域实际上意味着提供更多的初始状态。要添加更多的初始状态，可以“左移”一些状态，例如，如果我们有另一个名为 `AllOk` 的初始状态：

```c++
BOOST_MSM_EUML_DECLARE_STATE_MACHINE((transition_table,
                                     init_ << Empty << AllOk ),
                                    player_)
```

你可能还记得 `BOOST_MSM_EUML_STATE` 和 `BOOST_MSM_EUML_DECLARE_STATE_MACHINE` 的签名，在属性之后，我们可以定义标志，就像在基本的 MSM 前端中一样。为此，我们有另一个“左移”语法，例如：

```c++
BOOST_MSM_EUML_STATE((no_action,no_action, attributes_ <<no_attributes_, 
                      /* flags */ configure_<< PlayingPaused << CDLoaded), 
                    Paused)
```

我们现在定义了 `Paused` 将获得两个标志：`PlayingPaused` 和 `CDLoaded`，这些标志是通过另一个宏定义的：

```c++
BOOST_MSM_EUML_FLAG(CDLoaded)
```

这对应于 `Paused` 在基本前端中的以下定义：

```c++
struct Paused : public msm::front::state<>
{ 
   typedef mpl::vector2<PlayingPaused,CDLoaded> flag_list; 
};
```

在幕后，你实际上得到的是一个 `mpl::vector2`。

注意：由于我们使用的是带有 4 个参数的 `BOOST_MSM_EUML_STATE` 表达式版本，我们需要告诉 eUML 我们不需要任何属性。类似于 `cout << endl`，我们需要使用 `attributes_ << no_attributes_` 语法。

你可以使用状态机的 `is_flag_active` 方法来检查标志。你也可以使用提供的辅助函数 `is_flag_`（返回一个布尔值）用于状态和转换行为。例如，在使用 eUML 的 iPod 实现中，你会找到以下转换：

```c++
ForwardPressed == NoForward + EastPressed[!is_flag_(NoFastFwd)]
```

该函数还有一个可选的第二个参数，即调用该函数的状态机。默认情况下，使用 `fsm_`（当前状态机），但你可以提供一个返回对另一个状态机引用的函子。

eUML 还支持在状态（状态机）定义中定义延迟事件。为此，我们可以重用标志语法。例如：

```c++
BOOST_MSM_EUML_STATE((Empty_Entry,Empty_Exit, attributes_ << no_attributes_,
                      /* deferred */ configure_<< play ),Empty)
```

`configure_` 的左移操作还负责延迟事件的处理。在 `configure_` 中左移一个标志，状态将获得一个标志；左移一个事件，它将变成一个延迟事件。这取代了基本前端的定义：

```c++
typedef mpl::vector<play> deferred_events;
```

在[本教程](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/OrthogonalDeferredEuml.cpp)中，`player` 定义了一个第二个正交区域，其初始状态为 `AllOk`。`Empty` 和 `Open` 状态也会延迟 `play` 事件。`Open`、`Stopped` 和 `Pause` 状态同样通过左移至 `configure_` 支持标志 `CDLoaded`。

在函子前端中，我们还可以在转换中延迟一个事件，从而实现条件延迟。这在 eUML 中也可以通过使用 `defer_` 命令来实现，正如[本教程](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/OrthogonalDeferredEuml.cpp)所示。你会找到以下转换：

```c++
Open + play / defer_
```

这是一个内部转换。暂时忽略它。有趣的是，当事件 `play` 被触发且 `Open` 状态处于活动状态时，该事件将被延迟。现在添加一个保护条件，你可以有条件地延迟事件，例如：

```c++
Open + play [ some_condition ] / defer_
```

这与我们使用函子前端时的做法类似。这意味着我们有相同的约束。使用 `defer_` 而不是在状态声明中定义延迟事件时，我们需要告知 MSM 该状态机中存在延迟事件。我们通过在状态机定义中使用 `configure_` 声明来实现这一点，并在其中左移 `deferred_events` 配置标志：

```c++
BOOST_MSM_EUML_DECLARE_STATE_MACHINE((transition_table,
                                      init_ << Empty << AllOk,
                                      Entry_Action, 
                                      Exit_Action, 
                                      attributes_ << no_attributes_,
                                      configure_<< deferred_events ),
                                    player_)
```

一个教程展示了这种可能性。

## Customizing a state machine / Getting more speed

我们刚刚了解了如何使用 `configure_` 来定义延迟事件或标志。我们还可以像使用其他前端一样，用它来配置状态机：

- `configure_ << no_exception`：禁用异常处理  
- `configure_ << no_msg_queue`：停用消息队列  
- `configure_ << deferred_events`：手动启用事件延迟  

如果不需要上述功能，停用前两项且不激活第三项可以显著提高状态机的事件分发速度。我们在 eUML 中的速度测试示例就为此进行了最佳性能配置。

**重要提示**：由于退出伪状态使用消息队列将事件转发出子状态机，因此包含退出伪状态的状态机不能使用 `no_message_queue` 选项。

## Completion / Anonymous transitions

匿名转换（参见 [UML 教程](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/ch02s02.html#uml-anonymous)）是没有命名事件的转换，因此当源状态变为活动状态时，如果保护条件允许，它们会立即触发。由于没有事件，在定义这种转换时，只需省略转换中的“+”部分（即事件），例如：

```c++
State3 == State4 [always_true] / State3ToState4
State4 [always_true] / State3ToState4 == State3
```

请查看[此示例](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/AnonymousTutorialEuml.cpp)，它使用 eUML 实现了[前面定义](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/ch03s02.html#anonymous-transitions)的状态机。

## Internal transitions

像其他两个前端一样，eUML 支持两种定义内部转换的方法：

- 在状态机的转换表中定义。在这种情况下，你需要指定源状态、事件、动作和保护条件，但不需要指定目标状态，eUML 会将其解释为内部转换。例如，以下定义了 `Open` 状态内部在 `open_close` 事件上的转换：

  ```c++
  Open + open_close [internal_guard1] / internal_action1
  ```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;还提供了[一个完整的示例](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/EumlInternal.cpp)。

- 在状态的 `internal_transition_table` 中定义。例如：

  ```c++
  BOOST_MSM_EUML_DECLARE_STATE((Open_Entry,Open_Exit),Open_def)
  struct Open_impl : public Open_def
  {
    BOOST_MSM_EUML_DECLARE_INTERNAL_TRANSITION_TABLE((
         open_close [internal_guard1] / internal_action1
    ))
  };
  ```

注意，由于我们已经处于 `Open` 的上下文中，因此无需重复说明转换起源于 `Open`。

该实现还展示了子状态机的额外优势，子状态机可以同时拥有标准的 `transition_table` 和 `internal_transition_table`（优先级更高）。这使得如果你决定将一个状态扩展为完整的子状态机时更加方便。与添加正交区域的标准方法相比，这种方式也稍快一些，因为如果事件被内部表接受，事件分发将不会继续传递到子区域。这使得事件分发的时间复杂度为 O(1)，而不是 O（区域数量）。

### Kleene(any) event)

与函子前端一样，eUML 支持“任意事件”的概念，但 `boost::any` 不是 eUML 的有效终结符。如果你需要一个任意事件，请使用 `msm::front::euml::kleene`，它继承自 `boost::any`。与使用 `boost::any` 相同的转换将变为：

```c++
State1 + kleene == State2
```

## Other state types

我们已经看到了 `build_state` 函数，它创建了一个简单的状态。同样，eUML 为其他类型的状态提供了其他状态构建宏：

- `BOOST_MSM_EUML_TERMINATE_STATE` 接受与 `BOOST_MSM_EUML_STATE` 相同的参数，并定义一个终止状态。

- `BOOST_MSM_EUML_INTERRUPT_STATE` 接受与 `BOOST_MSM_EUML_STATE` 相同的参数，并定义一个中断状态。然而，表达式参数必须将结束中断的事件作为第一个元素，例如：
  ```cpp
  BOOST_MSM_EUML_INTERRUPT_STATE(( end_error /*end interrupt event*/,ErrorMode_Entry,ErrorMode_Exit ),ErrorMode)
  ```

- `BOOST_MSM_EUML_EXIT_STATE` 接受与 `BOOST_MSM_EUML_STATE` 相同的参数，并定义一个退出伪状态。然而，表达式参数必须将从退出点传播的事件作为第一个元素：
  ```cpp
  BOOST_MSM_EUML_EXIT_STATE(( event6 /*propagated event*/,PseudoExit1_Entry,PseudoExit1_Exit ),PseudoExit1)
  ```

- `BOOST_MSM_EUML_EXPLICIT_ENTRY_STATE` 定义一个进入伪状态。它接受三个参数：要进入的区域索引被定义为一个 `int` 参数，随后是类似于 `BOOST_MSM_EUML_STATE` 的配置表达式以及状态名称。因此，`BOOST_MSM_EUML_EXPLICIT_ENTRY_STATE(0 /*区域索引*/, (SubState2_Entry, SubState2_Exit), SubState2)` 定义了一个进入子状态机第一个区域的进入状态。

- `BOOST_MSM_EUML_ENTRY_STATE` 定义一个进入伪状态。它接受三个参数：要进入的区域索引被定义为一个 `int` 参数，随后是类似于 `BOOST_MSM_EUML_STATE` 的配置表达式以及状态名称。因此，`BOOST_MSM_EUML_ENTRY_STATE(0, (PseudoEntry1_Entry, PseudoEntry1_Exit), PseudoEntry1)` 定义了一个进入子状态机第一个区域的伪进入状态。

为了在转换表中使用这些状态，eUML 提供了 `explicit_`、`exit_pt_` 和 `entry_pt_` 函数。例如，从 `SubFsm2` 直接进入子状态 `SubState2` 可以表示为：

```c++
explicit_(SubFsm2,SubState2) == State1 + event2
```

由于分叉（Forks）是直接进入状态的列表，eUML 支持逻辑语法 `(state1, state2, ...)`，例如：

```c++
(explicit_(SubFsm2,SubState2), 
 explicit_(SubFsm2,SubState2b),
 explicit_(SubFsm2,SubState2c)) == State1 + event3 
```

进入点的进入语法与显式进入相同：

```c++
entry_pt_(SubFsm2,PseudoEntry1) == State1 + event4
```

对于退出点，语法与进入点相同，只不过退出点被用作转换的源状态：

```c++
State2 == exit_pt_(SubFsm2,PseudoExit1) + event6
```

入口教程同样适用于 eUML。

## Helper functions

我们已经看到了一些辅助工具，但还有更多，因此让我们进行更完整的描述：

- `event_`：在任何动作中使用，触发转换的事件
- `state_`：在进入和退出动作中使用，进入或退出的状态
- `source_`：在转换动作中使用，源状态
- `target_`：在转换动作中使用，目标状态
- `fsm_`：在任何动作中使用，处理转换的（最深层级的）状态机
- 这些对象也可以用作函数并返回一个属性，例如 `event_(cd_name)`
- `Int_<int value>`：表示一个 `int` 的函子
- `Char_<value>`：表示一个 `char` 的函子
- `Size_t_<value>`：表示一个 `size_t` 的函子
- `True_` 和 `False_` 函子分别返回 `true` 和 `false`
- `String_<mpl::string>`（Boost >= 1.40）：表示一个字符串的函子
- `if_then_else_(guard, action, action)` 其中 `action` 可以是一个动作序列
- `if_then_(guard, action)` 其中 `action` 可以是一个动作序列
- `while_(guard, action)` 其中 `action` 可以是一个动作序列
- `do_while_(guard, action)` 其中 `action` 可以是一个动作序列
- `for_(action, guard, action, action)` 其中 `action` 可以是一个动作序列
- `process_(some_event [, some state machine] [, some state machine] [, some state machine] [, some state machine])` 将调用当前状态机或作为第2、3、4、5个参数传递的状态机上的 `process_event(some_event)`。这允许向多个外部状态机发送事件
- `process_(event_)`：重新处理触发转换的事件
- `reprocess_()`：与上面相同，但写法更简洁
- `process2_(some_event, Value [, some state machine] [, some state machine] [, some state machine])` 将调用当前状态机或作为第3、4、5个参数传递的状态机上的 `process_event(some_event(Value))`
- `is_flag_(some_flag[, some state machine])` 将调用当前状态机或作为第2个参数传递的状态机上的 `is_flag_active`
- `Predicate_<some predicate>`：用于 STL 算法。将一元/二元函数包装为 eUML 兼容形式，以便它们可以在 STL 算法中使用

这可以非常有趣。例如，

```c++
/( if_then_else_(--fsm_(m_SongIndex) > Int_<0>(),/*if clause*/
                 show_playing_song, /*then clause*/
                 (fsm_(m_SongIndex)=Int_<1>(),process_(EndPlay))/*else clause*/
                 ) 
  )
```

意味着：`if (fsm.SongIndex > 0, call show_playing_song else {fsm.SongIndex=1; process EndPlay on fsm;}`

一些示例使用了这些功能：

- 在 BoostCon09 上介绍的 iPod 示例已经用 eUML [重写](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/iPodEuml.cpp)（编译器较弱的用户请跳过...）
- 在 BoostCon09 上介绍的 iPodSearch 示例也已用 eUML [重写](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/iPodSearchEuml.cpp)。在这个示例中，你还会找到一些 STL 函子使用的例子。
- [一个更简单的计时器](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/SimpleTimer.cpp)示例是一个很好的起点。

不幸的是，有一个小问题。使用 `MSM_EUML_METHOD` 或 `MSM_EUML_FUNCTION` 定义一个函子会创建一个正确的函子。你自己编写的 eUML 函子（如本节开头所述）也能很好地工作，但目前不支持与 `while_`、`if_then_` 和 `if_then_else_` 函数一起使用。

## Phoenix-like STL support

eUML 支持大多数 C++ 运算符（除了取地址运算符）。例如，可以编写 `event_(some_attribute)++` 或 `[source_(some_bool) && fsm_(some_other_bool)]`。但是，程序员在日常编程中需要的不仅仅是运算符。STL 显然是必不可少的。因此，eUML 提供了许多函子，以进一步减少在转换表中使用自定义函子的需求。对于几乎每一个 STL 算法或容器方法，都有一个对应的 eUML 函数定义。像 Boost.Phoenix 一样，对象上的“.”和“->”调用被替换为函数式编程范式，例如：

- `begin_(container)`，`end_(container)`：返回容器的迭代器。
- `empty_(container)`：返回 `container.empty()`
- `clear_(container)`：执行 `container.clear()`
- `transform_`：对应于 `std::transform`

简而言之，几乎所有 STL 方法或算法都有一个对应的函子，这些函子可以在转换表或状态动作中使用。参考文档列出了所有 eUML 函数及其底层函子（因此这种可能性不仅限于 eUML，也适用于基于函子的前端）。这个类似 Phoenix 的库的文件结构与 Boost.Phoenix 的文件结构相匹配。所有用于 STL 算法的函子都可以在以下路径找到：

```c++
#include <msm/front/euml/algorithm.hpp>
```

这些算法也被划分为多个子头文件，为了简化起见，其结构与 Phoenix 的结构相匹配：

```c++
#include < msm/front/euml/iteration.hpp> 
#include < msm/front/euml/transformation.hpp>
#include < msm/front/euml/querying.hpp> 
```

容器方法可以在以下路径找到：

```c++
#include < msm/front/euml/container.hpp>
```

或者可以简单地包含整个 STL 支持（你还需要包含 `euml.hpp`）：

```c++
#include < msm/front/euml/stl.hpp>
```

一些示例（可以在[本教程](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/iPodSearchEuml.cpp)中找到）：

- `push_back_(fsm_(m_tgt_container),event_(m_song))`：状态机有一个类型为 `std::vector<OneSong>` 的属性 `m_tgt_container`，而事件有一个类型为 `OneSong` 的属性 `m_song`。因此，这行代码将 `m_song` 添加到 `m_tgt_container` 的末尾。
- `if_then_( state_(m_src_it) != end_(fsm_(m_src_container)), process2_(OneSong(),*(state_(m_src_it)++)) )`：当前状态有一个属性 `m_src_it`（一个迭代器）。如果该迭代器不等于 `fsm.m_src_container.end()`，则在 `fsm` 上处理 `OneSong`，该事件通过复制构造的方式从 `state.m_src_it` 创建，同时我们对 `m_src_it` 进行后置递增。

## Writing actions with Boost.Phoenix (in development)

也可以使用缩减版的 Boost.Phoenix 功能编写动作、保护条件、状态进入和退出动作。此功能仍处于开发阶段，因此你可能会遇到一些意外情况。然而，简单的用例应该可以正常工作。不能混合使用 eUML 和 Phoenix 函子。不过，在一种语言中编写保护条件，在另一种语言中编写动作是可以的。

Phoenix 支持的语法比 eUML 可能支持的要大得多，因此你只能使用 Phoenix 语法的一个缩减集。这是由于 eUML 的性质决定的。运行时转换表定义被翻译成一个类型，使用 Boost.Typeof。结果是一个由函子类型组成的“普通”MSM 转换表。由于 C++ 不允许混合运行时和编译时构造，因此会有一些限制（尝试实例化模板类 `MyTemplateClass<i>` 其中 `i` 是一个 `int` 就会给你一个概念）。这意味着以下有效的 Phoenix 构造将无法工作：

- 字面量
- 函数指针
- 绑定 (`bind`)
- `->*`

MSM 还提供了占位符，这些占位符在其上下文中比 `arg1..argn` 更有意义：

- `_event`：触发转换的事件
- `_fsm`：处理事件的状态机
- `_source`：转换的源状态
- `_target`：转换的目标状态
- `_state`：对于状态进入/退出动作，进入/退出的状态

未来的 MSM 版本将更好地支持 Phoenix。你可以通过找出那些不工作但应该工作的案例来做出贡献，以便它们可以被添加进去。

Phoenix 支持默认是未激活的。要激活它，请在任何 MSM 头文件之前添加：`#define BOOST_MSM_EUML_PHOENIX_SUPPORT`

一个[简单的示例](https://www.boost.org/doc/libs/1_60_0/libs/msm/doc/HTML/examples/SimplePhoenix.cpp)展示了基本功能。
