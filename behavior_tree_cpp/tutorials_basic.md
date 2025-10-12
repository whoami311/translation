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
