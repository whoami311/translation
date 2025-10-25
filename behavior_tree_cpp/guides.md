# Guides

成为行为树大师

## Introduction to Scripting

行为树 4.X 引入了一个简单但强大的新概念：在 XML 中使用脚本语言。

这个脚本语言的语法很熟悉；它允许用户快速读取和写入黑板中的变量。

学习脚本工作原理的最简单方法是使用内置的 **Script** 动作，这在第二个教程中已经介绍过。

### Assignment operators, strings and numbers

示例：

```
param_A := 42
param_B = 3.14
message = 'hello world'
```

- 第一行将数字 42 赋值给黑板条目 **param_A**。
- 第二行将数字 3.14 赋值给黑板条目 **param_B**。
- 第三行将字符串 "hello world" 赋值给黑板条目 **message**。

> TIP：
> 运算符 ":=" 与 "=" 的区别在于，前者如果黑板中不存在该条目，会创建一个新的条目；而后者如果黑板中不存在该条目，则会抛出异常。

你也可以使用**分号**在同一个脚本中添加多个命令。

```
A:= 42; B:=24
```

#### Arithmetic operators and parenthesis

示例：

```
param_A := 7
param_B := 5
param_B *= 2
param_C := (param_A * 3) + param_B
```

生成的值为 `param_B` 为 10，`param_C` 为 31。

支持以下运算符：

| Operator | Assign Operator | Description |
| - | - | - |
| + | += | Add |
| - | -= | Subtract |
| * | *= | Multiply |
| / | /= | Divide |

注意，加法运算符是唯一也可以用于字符串的运算符（用于连接两个字符串）。

### Bitwise operator and hexadecimal numbers

这些运算符仅在值可以被转换为整数时有效。

将它们用于字符串或实数将导致异常。

示例：

```
value:= 0x7F
val_A:= value & 0x0F
val_B:= value | 0xF0
```

`val_A` 的值是 0x0F（即 15）；`val_B` 的值是 0xFF（即 255）。

| Binary Operators | Description |
| - | - |
| \|	| Bitwise or |
| &	| Bitwise and |
| ^	| Bitwise xor |

| Unary Operators | Description |
| - | - |
| ~ | Negate |

### Logic and comparison operators

返回布尔值的运算符。

示例：

```
val_A := true
val_B := 5 > 3
val_C := (val_A == val_B)
val_D := (val_A && val_B) || !val_C
```

| Operators	| Description |
| - | - |
| true/false | Booleans. Castable to 1 and 0 respectively |
| && | Logic and |
| || | Logic or |
| !	| Negation |
| == | Equality |
| != | Inequality |
| <	| Less |
| <= | Less equal |
| >	| Greater |
| >= | Greater equal |

#### Ternary operator if-then-else

示例：

```
val_B = (val_A > 1) ? 42 : 24
```

### C++ example

脚本语言演示，包括如何使用枚举来表示整数值。

XML 示例：

```xml
<root >
    <BehaviorTree>
        <Sequence>
            <Script code=" msg:='hello world' " />
            <Script code=" A:=THE_ANSWER; B:=3.14; color:=RED " />
            <Precondition if="A>B && color!=BLUE" else="FAILURE">
                <Sequence>
                  <SaySomething message="{A}"/>
                  <SaySomething message="{B}"/>
                  <SaySomething message="{msg}"/>
                  <SaySomething message="{color}"/>
                </Sequence>
            </Precondition>
        </Sequence>
    </BehaviorTree>
</root>
```

用于注册节点和枚举的 C++ 代码：

```c++
int main()
{
  // Simple tree: a sequence of two asynchronous actions,
  // but the second will be halted because of the timeout.

  BehaviorTreeFactory factory;
  factory.registerNodeType<SaySomething>("SaySomething");

  enum Color { RED=1, BLUE=2, GREEN=3 };
  // We can add these enums to the scripting language
  factory.registerScriptingEnums<Color>();

  // Or we can do it manually
  factory.registerScriptingEnum("THE_ANSWER", 42);

  auto tree = factory.createTreeFromText(xml_text);
  tree.tickWhileRunning();
  return 0;
}
```

预期输出：

```
Robot says: 42.000000
Robot says: 3.140000
Robot says: hello world
Robot says: 1.000000
```

注意，在底层，ENUM 总是被解释为其数值。

## Pre and Post conditions

利用前一教程中介绍的脚本语言的能力，BT.CPP 4.x 引入了前置条件（Pre）和后置条件（Post）的概念，即可以在节点实际 `tick()` 调用之前或之后运行的脚本。

前置条件和后置条件被**所有**节点支持，不需要对你的 C++ 代码做任何修改。

> CAUTION：脚本的目标**不**是编写复杂的代码，而只是为了提高行为树的可读性，并在非常简单的用例中减少对自定义 C++ 节点的需求。
> 如果你的脚本变得太长，可能需要重新考虑是否应该使用脚本。

### Pre conditions

| Name | Description | |
| - | - | - |
| **_skipIf** | 如果条件为真，则跳过该节点的执行 | |
| **_failureIf** | 如果条件为真，则跳过并返回 FAILURE | |
| **_successIf** | 如果条件为真，则跳过并返回 SUCCESS ||
| **_while** | 与 _skipIf 相同，但如果条件变为 false，也可能中断处于 RUNNING 的节点 ||

#### Example

在之前的教程中，我们展示了如何使用 fallback 在树中构建 if-then-else 逻辑。

新的语法要紧凑得多：

![pre conditions](img/pre_conditions.svg)

之前的方法：

```xml
<Fallback>
    <Inverter>
        <IsDoorClosed/>
    </Inverter>
    <OpenDoor/>
</Fallback>
```

如果不是使用自定义的 ConditionNode `IsDoorOpen`，而是将一个布尔值存储在名为 `door_closed` 的条目中，则可以将 XML 重写为：

```xml
<OpenDoor _skipIf="!door_closed"/>
```

### Post conditions

| Name | Description | |
| - | - | - |
| **_onSuccess** | 如果该节点返回 SUCCESS，则执行此脚本 | |
| **_onFailure** | 如果该节点返回 FAILURE，则执行此脚本 | |
| **_post** | 如果该节点返回 SUCCESS 或 FAILURE，则执行此脚本 ||
| **_onHalted** | 如果一个处于 RUNNING 状态的节点被中断（halted），则执行此脚本 ||

#### Example

在关于 Subtree 的教程中，我们看到过一个基于 **MoveBase** 的结果来写入特定黑板变量的逻辑。

在左侧，你可以看到这个逻辑在 BT.CPP 3.x 中是如何实现的，而使用 post conditions 后会简单得多。此外，新语法还支持使用**枚举**（enum）。

![post example](img/post_example.svg)

旧版本：

```xml
<Fallback>
    <Sequence>
        <MoveBase  goal="{target}"/>
        <SetBlackboard output_key="result" value="0" />
    </Sequence>
    <ForceFailure>
        <SetBlackboard output_key="result" value="-1" />
    </ForceFailure>
</Fallback>
```

新实现：

```xml
<MoveBase goal="{target}" 
          _onSuccess="result:=OK"
          _onFailure="result:=ERROR"/>
```

### Design pattern: error codes

与有限状态机相比，行为树在某些模式下可能表现不佳，尤其是当需要根据某个动作的不同结果执行不同策略时。

由于行为树的返回值仅限于 SUCCESS 和 FAILURE，这可能会令人难以直观表达。

一种解决方法是将 **结果/错误码** 存入黑板，但在 3.X 版本中这很繁琐。

前置条件（Pre conditions）可以帮助我们实现更具可读性的代码，例如：

![error codes](img/error_codes.svg)

在上图的树中，我们向 **MoveBase** 添加了一个输出端口 **return**，并根据 `error_code` 的值有条件地选择 Sequence 的第二个或第三个分支。

### Design pattern: states and declarative trees

尽管行为树承诺让我们摆脱状态的束缚，但事实是有时在没有状态的情况下很难推理我们的应用程序。

使用状态可以让我们的树更简单。例如，只有当机器人（或某个子系统）处于特定状态时，我们才会走树中的某个分支。

考虑这个节点及其前置/后置条件：

![states and declarative trees](img/states_and_declarative_trees.svg)

该节点仅在 state 等于 **DO_LANDING** 时执行；一旦 `altitude` 的值足够小，state 将被更改为 **LANDED**。

注意 DO_LANDING 和 LANDED 是枚举（enums），不是字符串。

> TIP：
> 这种模式的一个意想不到的副作用是，我们的节点变得更加**声明式**，也就是说，更容易将这个特定的节点/子树移动到树的其他部分。

## Asynchronous Actions

在设计响应式行为树时，理解两个主要概念非常重要：

- 我们所说的“**异步**（Asynchronous）”动作与“**同步**（Synchronous）”动作之间的区别。
- 在 BT.CPP 环境下，**并发**（Concurrency）与**并行**（Parallelism）之间的差异。

### Concurrency vs Parallelism

如果你在谷歌上搜索这些词，你会找到许多关于这个主题的优秀文章。

> INFO：
> **并发**（Concurrency）是指两个或更多任务可以在时间上重叠地开始、执行和完成。这并不一定意味着它们会在同一时刻同时运行。
> **并行**（Parallelism）是指任务在不同线程上同时运行，例如在多核处理器上。

BT.CPP 会**并发**执行所有节点。换句话说：

- 树的执行引擎是**单线程**的。
- 所有的 `tick()` 方法是**按顺序**执行的。
- 如果某个 `tick()` 方法是阻塞的，整个执行流程都会被阻塞。

我们通过“并发”和异步执行来实现响应式行为。

换句话说，一个执行时间较长的 Action 应该尽快返回状态 RUNNING。

这会告诉树的执行器，该 Action 已经开始执行，需要更多时间才能返回 SUCCESS 或 FAILURE。我们需要再次对该节点进行 tick，以确定状态是否发生了变化（轮询）。

异步节点可以将这个长时间执行的任务委托给另一个进程（使用进程间通信）或另一个线程。
