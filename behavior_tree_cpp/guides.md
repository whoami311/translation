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
