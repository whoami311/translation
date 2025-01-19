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

该类型的 `eval()` 函数用于评估此元素的数值。接下来，我们可以创建一个用于存储整数值（如 1、5 或 42）的元素：

```c++
struct Integer : Element
{
  int value;

  explicit Integer(const int value)
    : value(value) {}

  int eval() const override { return value; }
};
```

如果我们没有整数，那么我们必须有一个操作，比如加法或减法。在我们的情况下，所有操作都是二元的，意味着它们有两个部分。例如，在我们的模型中，`2+3` 可以用伪代码表示为 `BinaryOperation{Literal{2}, Literal{3}, addition}`：

```c++
struct BinaryOperation : Element
{
  enum Type { addition, subtraction } type;
  shared_ptr<Element> lhs, rhs;

  int eval() const override
  {
    if (type == addition)
      return lhs->eval() + rhs->eval();
    return lhs->eval() - rhs->eval();
  }
};
```

请注意，在前述内容中，我使用了一个普通的枚举（`enum`）而不是枚举类（`enum class`），这样我可以在后面写 `BinaryOperation::addition`。

但无论如何，现在进入解析过程。我们所需要做的就是将一个符号序列转换为表达式的二叉树。从一开始，它可能看起来如下：

```c++
shared_ptr<Element> parse(const vector<Token>& tokens)
{
  auto result = make_unique<BinaryOperation>();
  bool have_lhs = false; // this will need some explaining :)
  for (size_t i = 0; i < tokens.size(); i++)
  {
    auto token = tokens[i];
    switch(token.type)
    {
      // process each of the tokens in turn
    }
  }
  return result;
}
```

前述代码中我们唯一需要讨论的是 `have_lhs` 变量。请记住，您试图构建的是一棵树，而在树的根部我们期望有一个 `BinaryExpression`，根据定义，它有左和右两边。但是当我们处于一个数字上时，我们如何知道它是表达式的左边还是右边呢？没错，我们并不知道，这就是为什么我们要跟踪这一点。

现在让我们逐一分析这些情况。首先，整数——这些直接映射到我们的 `Integer` 构造，所以我们需要做的就是将文本转换为数字。（顺便说一句，如果我们想的话，也可以在词法分析阶段完成这一步。）

```c++
case Token::integer:
{
  int value = boost::lexical_cast<int>(token.text);
  auto integer = make_shared<Integer>(value);
  if (!have_lhs) {
    result->lhs = integer;
    have_lhs = true;
  }
  else result->rhs = integer;
}
```

加号和减号符号仅仅确定我们当前正在处理的操作类型，因此它们很简单：

```c++
case Token::plus:
  result->type = BinaryOperation::addition;
  break;
case Token::minus:
  result->type = BinaryOperation::subtraction;
  break;
```

然后是左括号。是的，只是左括号，在这个层次上我们并不显式检测右括号。基本思想很简单：找到匹配的右括号（我暂时忽略嵌套括号），提取整个子表达式，递归地解析它，并将其设置为当前处理的表达式的左边或右边：

```c++
case Token::lparen:
{
  int j = i;
  for (; j < tokens.size(); ++j)
    if (tokens[j].type == Token::rparen)
      break; // found it!

  vector<Token> subexpression(&tokens[i + 1], &tokens[j]);
  auto element = parse(subexpression); // recursive call
  if (!have_lhs)
  {
    result->lhs = element;
    have_lhs = true;
  }
  else result->rhs = element;
  i = j; // advance
}
```

在实际场景中，您会希望在这里加入更多的安全特性：不仅处理嵌套括号（我认为这是必须的），还要处理缺少闭合括号的不正确表达式。如果确实缺少闭合括号，您会如何处理？抛出异常？尝试解析剩余的部分并假设闭合括号在最后？还是其他方法？所有这些问题都留作读者练习。

从 C++ 的经验我们知道，为解析错误生成有意义的错误信息是非常困难的。实际上，您会发现一种称为“跳过”的现象，即在遇到不确定情况时，词法分析器或解析器将尝试跳过不正确的代码，直到遇到有意义的内容为止：这种做法正是静态分析工具所采用的，这些工具需要在用户输入不完整代码时也能正确工作。

## Using Lexer and Parser

随着 `lex()` 和 `parse()` 都已实现，我们终于可以解析表达式并计算其值：

```c++
string input{ "(13-4)-(12+1)" };
auto tokens = lex(input);
auto parsed = parse(tokens);
cout << input << " = " << parsed->eval() << endl;
// prints "(13-4)-(12+1) = -4"
```

## Parsing with Boost.Spirit

在现实世界中，几乎没有人会为复杂的东西手动编写解析器。当然，如果您正在解析像 XML 或 JSON 这样的“简单”数据存储格式，手动编写解析器是容易的。但如果您正在实现自己的领域特定语言（DSL）或编程语言，这就不现实了。

Boost.Spirit 是一个通过提供简洁（尽管不一定直观）的 API 来帮助创建解析器的库。该库并不试图显式分离词法分析和解析阶段（除非您真的想要这样做），而是允许您定义文本构造如何映射到您定义的类型对象上。

让我用 Tlön 编程语言展示一些使用 Boost.Spirit 的例子：

## Abstract Syntax Tree

首先，您需要构建抽象语法树（AST）。在这方面，我简单地创建一个支持访问者（Visitor）设计模式的基类，因为遍历这些结构非常重要：

```c++
struct ast_element
{
  virtual ~ast_element() = default;
  virtual void accept(ast_element_visitor& visitor) = 0;
};
```

然后，这个接口被用于定义我的语言中的不同代码构造，例如：

```c++
struct property : ast_element
{
  vector<wstring> names;
  type_specification type;
  bool is_constant{ false };
  wstring default_value;

  void accept(ast_element_visitor& visitor) override
  {
    visitor.visit(*this);
  }
};
```

前述属性的定义包含四个不同的部分，每个部分都存储在公共可访问的字段中。请注意，它使用了 `type_specification`，而 `type_specification` 本身是另一个 `ast_element`。

抽象语法树（AST）中的每一个类都需要适应 Boost.Fusion——这是另一个支持编译时（元编程）和运行时算法融合的 Boost 库。适应代码相当简单：

```c++
BOOST_FUSION_ADAPT_STRUCT(
  tlön::property,
  (std::vector<std::wstring>, names),
  (tlön::type_specification, type),
  (bool, is_constant),
  (std::wstring, default_value)
)
```

Spirit 在解析到常见的数据类型如 `std::vector` 或 `std::optional` 时没有任何问题。它在处理多态性方面有一些困难：与其让您的 AST 类型相互继承，Spirit 更倾向于您使用变体（variant），也就是说：

```c++
typedef variant<function_body, property, function_signature> class_member;
```

## Parser

Boost.Spirit 让我们可以将解析器定义为一组规则。所使用的语法非常类似于正则表达式或 BNF（Bachus-Naur Form）表示法，区别在于操作符放在符号之前，而不是之后。以下是一个示例规则：

```c++
class_declaration_rule %=
  lit(L"class ") >> +(alnum) % '.'
  >> -(lit(L"(") >> -parameter_declaration_rule % ',' >> lit(")"))
  >> "{"
  >> *(function_body_rule | property_rule | function_signature_rule)
  >> "}";
```

前述内容期望一个类声明以单词 `class` 开始。然后它期望一个或多个由点 `.` 分隔的单词（每个单词由一个或多个字母数字字符组成，因此使用 `+(alnum)` 表示），这里 `%` 操作符用于表示分隔。结果，如您所猜测的，将映射到一个 `vector`。随后，在大括号之后，我们期望零个或多个函数、属性或函数签名的定义——这些字段映射到的对应于我们之前使用 `variant` 定义的内容。

当然，某个元素是整个 AST 元素层次结构的“根”。在我们的情况下，这个根被称为文件（令人惊讶吧！），以下是一个既解析文件又对其进行漂亮打印的函数：

```c++
template<typename TLanguagePrinter, typename Iterator>
wstring parse(Iterator first, Iterator last)
{
  using spirit::qi::phrase_parse;

  file f;
  file_parser<wstring::const_iterator> fp{};
  auto b = phrase_parse(first, last, fp, space, f);
  if (b)
  {
    return TLanguagePrinter{}.pretty_print(f);
  }
  return wstring(L"FAIL");
}
```

前述内容中的类型 `TLanguagePrinter` 实际上是一个访问者（visitor），它知道如何将我们的抽象语法树（AST）渲染成另一种语言，比如 C++。

## Printer

解析了语言之后，我们可能想要编译它，或者在我的情况下，将其转换（transpile）成另一种语言。考虑到我们已经在整个 AST 层次结构中实现了 `accept()` 方法，这相对容易。

唯一的挑战是如何处理变体类型，因为这些需要特殊的访问者。对于 `std::variant`，您需要的是 `std::visit()`，但因为我们使用的是 `boost::variant`，所以应该调用 `boost::apply_visitor()` 函数。这个函数要求您提供一个继承自 `static_visitor` 的类的实例，并为每种可能的类型重载函数调用运算符。以下是一个示例：

```c++
struct default_value_visitor : static_visitor<>
{
  cpp_printer& printer;

  explicit default_value_visitor(cpp_printer& printer)
    : printer{printer}
  {
  }

  void operator()(const basic_type& bt) const
  {
    // for a scalar value, we just dump its default
    printer.buffer << printer.default_value_for(bt.name);
  }

  void operator()(const tuple_signature& ts) const
  {
    for (auto& e : ts.elements)
    {
      this->operator()(e.type);
      printer.buffer << ", ";
    }
    printer.backtrack(2);
  }
};
```

然后您会调用 `accept_visitor(foo, default_value_visitor{...})`，根据变体中实际存储的对象类型，正确的重载函数将会被调用。

##  Summary

首先需要说明的是，相对来说，解释器设计模式某种程度上不常见——构建解析器的挑战在今天被认为是非本质的，这就是为什么我看到它正从许多英国大学（包括我自己的）的计算机科学课程中被移除。此外，除非您计划从事语言设计或制作静态代码分析工具等工作，否则构建解析器的技能不太可能有很高的需求。

话虽如此，解释的挑战是计算机科学中的一个完全独立的领域，单本设计模式书籍的一个章节无法充分涵盖这一领域的全部内容。如果您对这个主题感兴趣，我建议您查看专门为词法分析器  /解析器构造设计的框架，如 Lex / Yacc、ANTLR 以及其他许多类似工具。我还推荐为流行的集成开发环境（IDE）编写静态分析插件——这是了解真实抽象语法树（AST）的样子、如何遍历以及甚至修改它们的好方法。
