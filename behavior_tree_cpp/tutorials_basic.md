# Tutorials (Basic)

用 5 分钟时间来了解 Docusaurus 中最重要的核心概念。

## 01. Your first Behavior Tree

行为树（Behavior Trees）与状态机（State Machines）类似，本质上只是一种在合适的时间、满足特定条件时调用**回调函数**的机制。至于这些回调内部执行什么操作，则完全由你决定。

在本系列教程中，我们将交替使用 **“调用回调（to invoke the callback）”** 和 **“tick”** 这两个表达，它们的含义相同。

在教程的大多数示例中，我们的示例性动作（dummy Actions）只是简单地在控制台打印一些信息，但请记住，真正的“生产级”代码通常会执行更复杂的操作。

接下来，我们将创建这样一棵简单的树：

![first behavior tree](img/first_behavior_tree.svg)

### How to create your own ActionNodes

创建 TreeNode 的默认方式（也是推荐方式）是通过继承（inheritance）来实现。

```c++
// Example of custom SyncActionNode (synchronous action)
// without ports.
class ApproachObject : public BT::SyncActionNode
{
public:
  ApproachObject(const std::string& name) :
      BT::SyncActionNode(name, {})
  {}

  // You must override the virtual function tick()
  BT::NodeStatus tick() override
  {
    std::cout << "ApproachObject: " << this->name() << std::endl;
    return BT::NodeStatus::SUCCESS;
  }
};
```

如你所见：

- 每个 TreeNode 实例都有一个 `name`。这个标识符是供人类阅读的，**不**需要是唯一的。
- 方法 `tick()` 是执行实际 Action 的地方。它必须始终返回一个 `NodeStatus`，即 RUNNING、SUCCESS 或 FAILURE。

或者，我们也可以通过**依赖注入**（dependency injection）的方式，使用函数指针（即 “functor”）来创建一个 **TreeNode**。

该 functor 必须具有以下函数签名：

```c++
BT::NodeStatus myFunction(BT::TreeNode& self) 
```

例如：

```c++
using namespace BT;

// Simple function that return a NodeStatus
BT::NodeStatus CheckBattery()
{
  std::cout << "[ Battery: OK ]" << std::endl;
  return BT::NodeStatus::SUCCESS;
}

// We want to wrap into an ActionNode the methods open() and close()
class GripperInterface
{
public:
  GripperInterface(): _open(true) {}
    
  NodeStatus open() 
  {
    _open = true;
    std::cout << "GripperInterface::open" << std::endl;
    return NodeStatus::SUCCESS;
  }

  NodeStatus close() 
  {
    std::cout << "GripperInterface::close" << std::endl;
    _open = false;
    return NodeStatus::SUCCESS;
  }

private:
  bool _open; // shared information
};
```

我们可以基于以下任意一个 functor 构建一个 `SimpleActionNode`：

- `CheckBattery()`
- `GripperInterface::open()`
- `GripperInterface::close()`

### Create a tree dynamically with an XML

让我们来看下面这个名为 **my_tree.xml** 的 XML 文件：

```xml
 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
            <CheckBattery   name="check_battery"/>
            <OpenGripper    name="open_gripper"/>
            <ApproachObject name="approach_object"/>
            <CloseGripper   name="close_gripper"/>
        </Sequence>
     </BehaviorTree>
 </root>
```

TIP：你可以在此处找到关于 XML 模式（schema）的更多详细信息。

我们必须先将自定义 TreeNodes 注册到 `BehaviorTreeFactory` 中，然后再从文件或文本中加载 XML。

XML 中使用的标识符必须与注册 TreeNodes 时使用的标识符一致。

属性 "name" 表示该实例的名称，**它是可选的**。

```c++
#include "behaviortree_cpp/bt_factory.h"

// file that contains the custom nodes definitions
#include "dummy_nodes.h"
using namespace DummyNodes;

int main()
{
    // We use the BehaviorTreeFactory to register our custom nodes
  BehaviorTreeFactory factory;

  // The recommended way to create a Node is through inheritance.
  factory.registerNodeType<ApproachObject>("ApproachObject");

  // Registering a SimpleActionNode using a function pointer.
  // You can use C++11 lambdas or std::bind
  factory.registerSimpleCondition("CheckBattery", [&](TreeNode&) { return CheckBattery(); });

  //You can also create SimpleActionNodes using methods of a class
  GripperInterface gripper;
  factory.registerSimpleAction("OpenGripper", [&](TreeNode&){ return gripper.open(); } );
  factory.registerSimpleAction("CloseGripper", [&](TreeNode&){ return gripper.close(); } );

  // Trees are created at deployment-time (i.e. at run-time, but only 
  // once at the beginning). 
    
  // IMPORTANT: when the object "tree" goes out of scope, all the 
  // TreeNodes are destroyed
   auto tree = factory.createTreeFromFile("./my_tree.xml");

  // To "execute" a Tree you need to "tick" it.
  // The tick is propagated to the children based on the logic of the tree.
  // In this case, the entire sequence is executed, because all the children
  // of the Sequence return SUCCESS.
  tree.tickWhileRunning();

  return 0;
}

/* Expected output:
*
  [ Battery: OK ]
  GripperInterface::open
  ApproachObject: approach_object
  GripperInterface::close
*/
```

## 02. Blackboard and ports

正如我们之前所解释的，自定义 TreeNodes 可以用来执行任意简单或复杂的软件功能。它们的目标是提供一个**更高层次的抽象接口**。

因此，从概念上讲，它们与**函数**并无本质区别。

类似于函数，我们通常希望：

- 向节点传递参数/输入（**inputs**）；
- 从节点获取某种信息/输出（**outputs**）。
- 一个节点的输出可以作为另一个节点的输入。

BehaviorTree.CPP 提供了一种通过**端口**（ports）进行**数据流传递**的基础机制，这种机制既简单易用，又灵活且类型安全。

在本教程中，我们将创建如下的行为树：

![tutorial blackboard](img/tutorial_blackboard.svg)

Main Concepts：

- “Blackboard” 是一个简单的**键/值存储**，由树中所有节点共享。
- Blackboard 的一个 “条目（entry）” 就是一个**键/值对**。
- **输入端口**（Input port）可以读取 Blackboard 中的条目，而**输出端口**（Output port）可以写入条目。

### Inputs ports

一个有效的输入可以是：

* 节点将读取和解析的**静态字符串**；
* 指向 Blackboard 条目的“指针”，由**键**（key）标识。

假设我们想创建一个 ActionNode，名为 `SaySomething`，它应该在 `std::cout` 上打印一个指定的字符串。

为了传递这个字符串，我们将使用名为 **message** 的输入端口。

考虑以下几种可选的 XML 语法：

```xml
<SaySomething name="first"   message="hello world" />
<SaySomething name="second" message="{greetings}" />
```

- 在**第一个**节点中，端口接收到字符串 "hello world"；
- 而**第二个**节点则从 Blackboard 中查找名为 "greetings" 的条目值。

CAUTION：条目 "greetings" 的值可以在运行时发生变化（并且很可能会变化）。

ActionNode `SaySomething` 可以实现如下：

```c++
// SyncActionNode (synchronous action) with an input port.
class SaySomething : public SyncActionNode
{
public:
  // If your Node has ports, you must use this constructor signature 
  SaySomething(const std::string& name, const NodeConfig& config)
    : SyncActionNode(name, config)
  { }

  // It is mandatory to define this STATIC method.
  static PortsList providedPorts()
  {
    // This action has a single input port called "message"
    return { InputPort<std::string>("message") };
  }

  // Override the virtual function tick()
  NodeStatus tick() override
  {
    Expected<std::string> msg = getInput<std::string>("message");
    // Check if expected is valid. If not, throw its error
    if (!msg)
    {
      throw BT::RuntimeError("missing required input [message]: ", 
                              msg.error() );
    }
    // use the method value() to extract the valid message.
    std::cout << "Robot says: " << msg.value() << std::endl;
    return NodeStatus::SUCCESS;
  }
};
```

当自定义 TreeNode 拥有输入端口和/或输出端口时，这些端口必须在**静态**方法中进行声明：

```c++
static MyCustomNode::PortsList providedPorts();
```

可以使用模板方法 `TreeNode::getInput<T>(key)` 来读取端口 `message` 的输入。

该方法可能因多种原因而失败。用户需要自行检查返回值的有效性，并决定采取何种处理方式：

- 返回 `NodeStatus::FAILURE`？
- 抛出异常？
- 使用不同的默认值？

IMPORTANT：

**强烈**建议在 `tick()` 方法内部调用 `getInput()`，而**不要**在类的构造函数中调用。

C++ 代码应当预期输入的实际值会在**运行时发生变化**，因此需要定期更新该值。

### Output ports

指向 Blackboard 条目的输入端口只有在另一个节点已经向该条目写入“某些内容”之后才是有效的。

`ThinkWhatToSay` 就是一个示例节点，它使用**输出端口**将一个字符串写入 Blackboard 条目。

```c++
class ThinkWhatToSay : public SyncActionNode
{
public:
  ThinkWhatToSay(const std::string& name, const NodeConfig& config)
    : SyncActionNode(name, config)
  { }

  static PortsList providedPorts()
  {
    return { OutputPort<std::string>("text") };
  }

  // This Action writes a value into the port "text"
  NodeStatus tick() override
  {
    // the output may change at each tick(). Here we keep it simple.
    setOutput("text", "The answer is 42" );
    return NodeStatus::SUCCESS;
  }
};
```

或者，在大多数情况下，为了调试目的，可以使用内置动作 `Script` 向条目写入一个静态值。

```xml
<Script code=" the_answer:='The answer is 42' " />
```

我们将在 BT.CPP 中关于新脚本语言的教程中，详细讲解 Action **Script**。

TIP：如果你正在从 BT.CPP 3.X 版本迁移，**Script** 可以作为 **SetBlackboard** 的直接替代，而 **SetBlackboard** 现在已不推荐使用。

### A complete example

在这个示例中，会执行一个包含 3 个动作的 Sequence：

- Action 1 从一个静态字符串读取输入 `message`。
- Action 2 向 Blackboard 中名为 `the_answer` 的条目写入内容。
- Action 3 从 Blackboard 中名为 `the_answer` 的条目读取输入 `message`。

```xml
<root BTCPP_format="4" >
    <BehaviorTree ID="MainTree">
       <Sequence name="root_sequence">
           <SaySomething     message="hello" />
           <ThinkWhatToSay   text="{the_answer}"/>
           <SaySomething     message="{the_answer}" />
       </Sequence>
    </BehaviorTree>
</root>
```

用于注册并执行该行为树的 C++ 代码如下：

```c++
#include "behaviortree_cpp/bt_factory.h"

// file that contains the custom nodes definitions
#include "dummy_nodes.h"
using namespace DummyNodes;

int main()
{  
  BehaviorTreeFactory factory;
  factory.registerNodeType<SaySomething>("SaySomething");
  factory.registerNodeType<ThinkWhatToSay>("ThinkWhatToSay");

  auto tree = factory.createTreeFromFile("./my_tree.xml");
  tree.tickWhileRunning();
  return 0;
}

/*  Expected output:
  Robot says: hello
  Robot says: The answer is 42
*/
```

我们通过使用相同的键（`the_answer`）将输出端口“连接”到输入端口，换句话说，它们“指向” Blackboard 中同一个条目。

这些端口之所以可以相互连接，是因为它们的类型相同，即 `std::string`。如果尝试连接类型不同的端口，方法 `factory.createTreeFromFile` 将抛出异常。

## 03. Ports with generic types

在前面的教程中，我们介绍了输入端口和输出端口，其端口类型为 `std::string`。

接下来，我们将展示如何为端口分配通用的 C++ 类型。

### Parsing a string

BehaviorTree.CPP 支持将字符串自动转换为常见类型，例如 `int`、`long`、`double`、`bool`、`NodeStatus` 等。用户自定义类型也可以很容易地得到支持。

例如：

```c++
// We want to use this custom type
struct Position2D 
{ 
  double x;
  double y; 
};
```

为了让 XML 加载器能够从字符串实例化 `Position2D`，我们需要提供模板特化 `BT::convertFromString<Position2D>(StringView)`。

`Position2D` 如何序列化为字符串由你自行决定；在这个例子中，我们只是用*分号*分隔两个数字。

```c++
// Template specialization to converts a string to Position2D.
namespace BT
{
    template <> inline Position2D convertFromString(StringView str)
    {
        // We expect real numbers separated by semicolons
        auto parts = splitString(str, ';');
        if (parts.size() != 2)
        {
            throw RuntimeError("invalid input)");
        }
        else
        {
            Position2D output;
            output.x     = convertFromString<double>(parts[0]);
            output.y     = convertFromString<double>(parts[1]);
            return output;
        }
    }
} // end namespace BT
```

- `StringView` 是 C++11 中的 `std::string_view` 版本。你可以传入 `std::string` 或 `const char*`。
- 该库提供了一个简单的 `splitString` 函数，你也可以使用其他函数，例如 `boost::algorithm::split`。
- 我们可以使用特化的 `convertFromString<double>()`。

### Example

和前一节教程一样，我们可以创建两个自定义 Actions：一个向端口写入数据，另一个从端口读取数据。

```c++
class CalculateGoal: public SyncActionNode
{
  public:
    CalculateGoal(const std::string& name, const NodeConfig& config):
      SyncActionNode(name,config)
    {}

    static PortsList providedPorts()
    {
      return { OutputPort<Position2D>("goal") };
    }

    NodeStatus tick() override
    {
      Position2D mygoal = {1.1, 2.3};
      setOutput<Position2D>("goal", mygoal);
      return NodeStatus::SUCCESS;
    }
};

class PrintTarget: public SyncActionNode
{
  public:
    PrintTarget(const std::string& name, const NodeConfig& config):
        SyncActionNode(name,config)
    {}

    static PortsList providedPorts()
    {
      // Optionally, a port can have a human readable description
      const char*  description = "Simply print the goal on console...";
      return { InputPort<Position2D>("target", description) };
    }
      
    NodeStatus tick() override
    {
      auto res = getInput<Position2D>("target");
      if( !res )
      {
        throw RuntimeError("error reading port [target]:", res.error());
      }
      Position2D target = res.value();
      printf("Target positions: [ %.1f, %.1f ]\n", target.x, target.y );
      return NodeStatus::SUCCESS;
    }
};
```

现在我们可以像往常一样连接输入/输出端口，使它们指向 Blackboard 中同一个条目。

下一个示例中的行为树是一个包含 4 个动作的 Sequence：

- 使用动作 `CalculateGoal` 将一个 `Position2D` 值存入条目 **GoalPosition**。
- 调用 `PrintTarget`，其输入端口 “target” 将从 Blackboard 条目 **GoalPosition** 读取数据。
- 使用内置动作 `Script` 将字符串 "-1;3" 分配给键 **OtherGoal**。字符串到 `Position2D` 的转换将自动完成。
- 再次调用 `PrintTarget`，其输入端口 “target” 将从条目 `OtherGoal` 读取数据。

```c++
static const char* xml_text = R"(

 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root">
            <CalculateGoal goal="{GoalPosition}" />
            <PrintTarget   target="{GoalPosition}" />
            <Script        code=" OtherGoal:='-1;3' " />
            <PrintTarget   target="{OtherGoal}" />
        </Sequence>
     </BehaviorTree>
 </root>
 )";

int main()
{
  BT::BehaviorTreeFactory factory;
  factory.registerNodeType<CalculateGoal>("CalculateGoal");
  factory.registerNodeType<PrintTarget>("PrintTarget");

  auto tree = factory.createTreeFromText(xml_text);
  tree.tickWhileRunning();

  return 0;
}
/* Expected output:

    Target positions: [ 1.1, 2.3 ]
    Converting string: "-1;3"
    Target positions: [ -1.0, 3.0 ]
*/
```

## 04. Reactive and Asynchronous behaviors

下一个示例展示了 `SequenceNode` 与 `ReactiveSequence` 的区别。

我们将实现一个 **异步动作（Asynchronous Action）**，也就是一个需要较长时间完成的动作，在未满足完成条件时会返回 RUNNING。

异步动作具有以下要求：

- 在 `tick()` 方法中不应阻塞过长时间，执行流程应尽可能快速返回。
- 如果调用了 `halt()` 方法，应尽可能快速中止执行。

CAUTION：

了解更多关于 **异步动作（Asynchronous Actions）** 的内容

用户应当充分理解 BT.CPP 中的 并发机制（Concurrency），并学习开发自定义 异步动作（Asynchronous Actions）的最佳实践。你可以在这里找到一篇详细的文章。

### StatefulActionNode

**StatefulActionNode** 是实现异步动作（Asynchronous Actions）的首选方式。

当你的代码包含**请求-响应模式**时，它尤其有用——即动作向另一个进程发送异步请求，并周期性检查是否收到了回复。

根据收到的回复，它可能返回 SUCCESS 或 FAILURE。

如果不是与外部进程通信，而是执行耗时计算，你可以将计算拆分成小的“块”，或者将计算移到另一个线程中执行（参见 AsyncThreadedAction 教程）。

**StatefulActionNode** 的派生类必须重写以下虚方法，而不是 `tick()`：

- `NodeStatus onStart()`：当节点处于 IDLE 状态时调用。它可以立即成功或失败，也可以返回 RUNNING。如果返回 RUNNING，下次收到 tick 时会执行 `onRunning()`。
- `NodeStatus onRunning()`：当节点处于 RUNNING 状态时调用，返回新的状态。
- `void onHalted()`：当该节点被树中的其他节点中止时调用。

下面我们创建一个名为 **MoveBaseAction** 的示例节点：

```c++
// Custom type
struct Pose2D
{
    double x, y, theta;
};

namespace chr = std::chrono;

class MoveBaseAction : public BT::StatefulActionNode
{
  public:
    // Any TreeNode with ports must have a constructor with this signature
    MoveBaseAction(const std::string& name, const BT::NodeConfig& config)
      : StatefulActionNode(name, config)
    {}

    // It is mandatory to define this static method.
    static BT::PortsList providedPorts()
    {
        return{ BT::InputPort<Pose2D>("goal") };
    }

    // this function is invoked once at the beginning.
    BT::NodeStatus onStart() override;

    // If onStart() returned RUNNING, we will keep calling
    // this method until it return something different from RUNNING
    BT::NodeStatus onRunning() override;

    // callback to execute if the action was aborted by another node
    void onHalted() override;

  private:
    Pose2D _goal;
    chr::system_clock::time_point _completion_time;
};

//-------------------------

BT::NodeStatus MoveBaseAction::onStart()
{
  if ( !getInput<Pose2D>("goal", _goal))
  {
    throw BT::RuntimeError("missing required input [goal]");
  }
  printf("[ MoveBase: SEND REQUEST ]. goal: x=%f y=%f theta=%f\n",
         _goal.x, _goal.y, _goal.theta);

  // We use this counter to simulate an action that takes a certain
  // amount of time to be completed (200 ms)
  _completion_time = chr::system_clock::now() + chr::milliseconds(220);

  return BT::NodeStatus::RUNNING;
}

BT::NodeStatus MoveBaseAction::onRunning()
{
  // Pretend that we are checking if the reply has been received
  // you don't want to block inside this function too much time.
  std::this_thread::sleep_for(chr::milliseconds(10));

  // Pretend that, after a certain amount of time,
  // we have completed the operation
  if(chr::system_clock::now() >= _completion_time)
  {
    std::cout << "[ MoveBase: FINISHED ]" << std::endl;
    return BT::NodeStatus::SUCCESS;
  }
  return BT::NodeStatus::RUNNING;
}

void MoveBaseAction::onHalted()
{
  printf("[ MoveBase: ABORTED ]");
}
```

### Sequence VS ReactiveSequence
