# Interpret

解释器（Interpreter）设计模式的目标，正如您所猜测的那样，是对输入进行解释，特别是文本输入，尽管说实在的，它并不局限于文本。解释器的概念与编译原理等大学课程中教授的内容紧密相关。由于我们在这里没有足够的篇幅深入探讨各种解析器和其他复杂性的问题，本章的目的只是展示一些可能需要解释的例子。

以下是一些较为明显的情况：

- 数值字面量，如 `42` 或 `1.234e12`，需要被解释以高效地存储为二进制格式。在 C++ 中，这些操作既可以通过像 `atof()` 这样的 C API 来处理，也可以通过更复杂的库如 `Boost.LexicalCast` 来实现。
- 正则表达式帮助我们在文本中查找模式，但需要注意的是，正则表达式实际上是一种嵌入式的特定领域语言（DSL）。自然，在使用它们之前，必须正确地解释它们。
- 任何结构化数据，无论是 CSV、XML、JSON，还是更复杂的数据格式，在使用前都需要进行解释。
- 在解释器应用的巅峰，我们有完全成熟的编程语言。毕竟，像 C 或 Python 这样的语言的编译器或解释器必须在生成可执行文件之前真正理解该语言。

鉴于与解释相关的挑战的广泛性和多样性，我们将简单地看一些例子。这些例子旨在说明如何构建一个解释器：要么从零开始创建，要么利用帮助大规模实现这些功能的库。

## Numeric Expression Evaluator

让我们想象一下，我们决定解析非常简单的数学表达式，例如 `3+(5-4)`，也就是说，我们将自己限制在加法、减法和括号内。我们希望编写一个程序，它可以读取这样的表达式，并且当然，计算表达式的最终值。

我们将手动构建这个计算器，而不依赖任何解析框架。这应该能够突出解析文本输入所涉及的一些复杂性。

## Lexing

解释表达式的第一个步骤被称为词法分析（lexing），它涉及将字符序列转换为符号（tokens）序列。一个符号通常是原始的语法元素，最终我们应该得到这些符号的平坦序列。在我们的情况下，一个符号可以是：

- 一个整数
- 一个运算符（加号或减号）
- 一个开括号或闭括号

因此，我们可以定义如下结构：

```c++
struct Token
{
  enum Type { integer, plus, minus, lparen, rparen } type;
  string text;

  explicit Token(Type type, const string& text)
    : type{type}, text{text} {}

  friend ostream& operator<<(ostream& os, const Token& obj)
  {
    return os << "`" << obj.text << "`";
  }
};
```

您会注意到 `Token` 并不是一个枚举类型，因为除了类型之外，我们还想要存储与此符号相关的文本，因为它并不总是预定义的。

因此，现在给定一个包含表达式的 `std::string`，我们可以定义一个词法分析过程，该过程将文本转换为 `vector<Token>`：

```c++
vector<Token> lex(const string& input)
{
  vector<Token> result;

  for (int i = 0; i < input.size(); ++i)
  {
    switch (input[i])
    {
    case '+':
      result.push_back(Token{ Token::plus, "+" });
      break;
    case '-':
      result.push_back(Token{ Token::minus, "-" });
      break;
    case '(':
      result.push_back(Token{ Token::lparen, "(" });
      break;
    case ')':
      result.push_back(Token{ Token::rparen, ")" });
      break;
    default:
      // number ???
    }
  }
}
```

解析预定义的符号很容易。实际上，我们可以将它们添加为 `map<BinaryOperation, char>` 以简化事情。但是，解析数字并不那么容易。如果我们遇到了一个 '1'，我们应该等待并观察下一个字符是什么。为此，我们定义了一个单独的例程：

```c++
ostringstream buffer;
buffer << input[i];
for (int j = i + 1; j < input.size(); ++j)
{
  if (isdigit(input[j]))
  {
    buffer << input[j];
    ++i;
  }
  else
  {
    result.push_back(Token{ Token::integer, buffer.str() });
    break;
  }
}
```

实际上，当我们持续读取（抽取）数字时，我们将它们添加到缓冲区。完成后，我们根据整个缓冲区创建一个 `Token`，并将其添加到结果 `vector` 中。

## Parsing

解析的过程将符号序列转换为有意义的、通常是面向对象的结构。在最顶层，通常有一个抽象的父类型是有用的，所有树的元素都实现这个类型：

```c++
struct Element
{
  virtual int eval() const = 0;
};
```
