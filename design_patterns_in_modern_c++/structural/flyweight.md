# Flyweight

享元（Flyweight，也有时被称为令牌或 Cookie）是一个临时组件，它作为某种“智能引用”发挥作用。通常，当您有许多非常相似的对象，并且希望最小化存储这些值所占用的内存时，会使用享元。

让我们来看一些这种模式变得相关的场景。

## User Names

想象一下一款大型多人在线角色扮演游戏。我敢打赌有超过一个用户的名称是 John Smith——这仅仅是因为它是一个常见的名字。所以，如果我们一遍又一遍地存储这个名字（以 ASCII 格式），我们将为每个这样的用户花费 11 个字节。相反，我们可以只存储一次这个名字，然后为每个拥有这个名字的用户存储一个指针（这只需要 8 个字节）。这样可以节省不少空间。

进一步讲，将名字拆分为名和姓可能更有意义：这样一来，Fitzgerald Smith 将由两个指针表示（总共 16 个字节），分别指向名和姓。实际上，如果我们使用索引代替名字，可以大幅减少所使用的字节数。您不会期望有 2^64 个不同的名字吧？

我们实际上可以对此进行类型定义（typedef），以便日后调整：

```c++
typedef uint32_t key;
```

使用此定义，我们可以将用户定义为如下形式：

```c++
struct User
{     
  User(const string& first_name, const string& last_name)
    : first_name{add(first_name)}, last_name{add(last_name)} {}
 ...
protected:
  key first_name, last_name;
  static bimap<key, string> names;
  static key seed;
  static key add(const string& s) { ... }
};
```

如您所见，构造函数使用 `add()` 函数的返回结果来初始化成员 `first_name` 和 `last_name`。此函数根据需要将键值对（键是从种子生成的）插入到 `names` 结构中。这里我使用了一个 `boost::bimap`（双向映射），因为它更容易查找重复项——请记住，如果名或姓已经在 `bimap` 中，我们只是返回一个指向它的索引。

因此，以下是 `add()` 的实现：

```c++
static key add(const string& s)
{
  auto it = names.right.find(s);
  if (it == names.right.end())
  {
    // add it
    names.insert({++seed, s});
    return seed;
  }
  return it->second;
}
```

这是一个相当标准的 get-or-add（获取或添加）机制的实现。如果您之前没有接触过 bimap，您可能需要查阅 bimap 的文档以了解更多它是如何工作的。

现在，如果我们想要实际暴露名和姓（这些字段是 `protected`，并且类型为 `key`，不是特别有用！），我们可以提供相应的 `getter` 和 `setter` 方法：

```c++
const string& get_first_name() const
{
  return names.left.find(last_name)->second;
}

const string& get_last_name() const
{
  return names.left.find(last_name)->second;
}
```

例如，为了定义用户的流输出操作符，您可以简单地编写：

```c++
friend ostream& operator<<(ostream& os, const User& obj)
{
  return os
    << "first_name: " << obj.get_first_name()
    << " last_name: " << obj.get_last_name();
}
```

就是这样。我不会提供关于节省了多少空间的统计数据（这确实取决于您的样本大小），但希望很明显的是，在大量用户名重复的情况下，节省的空间是显著的——特别是如果您通过更改其 `typedef` 来进一步减少 `sizeof(key)` 的话。

##  Boost.Flyweight

在前面的例子中，我手工创建了一个享元（Flyweight），即使我可以重用 Boost 库中可用的一个。`boost::flyweight` 类型正如其名：构建一个节省空间的享元。

这使得 `User` 类的实现相当简单：

```c++
struct User2
{
  flyweight<string> first_name, last_name;

  User2(const string& first_name, const string& last_name)
    : first_name{first_name},
      last_name{last_name} {}
};
```

您可以通过运行以下代码来验证它实际上是一个享元（flyweight）：

```c++
User2 john_doe{ "John", "Doe" };
User2 jane_doe{ "Jane", "Doe" };
cout << boolalpha <<
   (&jane_doe.last_name.get() == &john_doe.last_name.get()); // true
```

##  String Ranges

如果您调用 `std::string::substring()`，是否应该返回一个全新构建的字符串？对此问题的答案并不统一：如果您想要操纵这个子串，那么创建一个新的字符串是合理的；但如果希望对子串的更改能够影响原始对象呢？一些编程语言（例如 Swift、Rust）明确地将子串作为范围返回，这再次是享元（Flyweight）模式的一种实现，它不仅节省了内存使用量，还允许我们通过范围来操纵底层对象。

C++ 中字符串范围的等价物是 `string_view`，对于数组也有类似的变体——任何可以避免复制数据的方法。让我们尝试构造我们自己的、非常简单的字符串范围。

假设我们有一个类中存储了一段文本，并且我们想要获取该文本的一个范围并将其大写化，就像文字处理软件或集成开发环境（IDE）可能会做的那样。我们可以直接将每个字母大写，但假设我们希望保持底层纯文本的原始状态不变，只在使用流输出操作符时才进行大写化。

## Naïve Approach

一个非常简单但不太高效的方法是定义一个布尔数组，其大小与纯文本字符串匹配，值表示是否需要将对应位置的字符大写。我们可以像这样实现它：

```c++
class FormattedText
{
  string plainText;
  bool *caps;
public:
  explicit FormattedText(const string& plainText)
    : plainText{plainText}
  {
    caps = new bool[plainText.length()];
  }

  ~FormattedText()
  {
    delete[] caps;
  }
};
```

我们现在可以为大写化特定范围创建一个实用方法：

```c++
void capitalize(int start, int end)
{
  for (int i = start; i <= end; ++i)
   caps[i] = true;
}
```

然后定义一个流输出操作符，利用布尔掩码来决定是否大写化字符：

```c++
friend std::ostream& operator<<(std::ostream& os, const FormattedText& obj)
{
  string s;
  for (int i = 0; i < obj.plainText.length(); ++i)
  {
    char c = obj.plainText[i];
    s += (obj.caps[i] ? toupper(c) : c);
  }
  return os << s;
}
```

别误会，这种方法是可行的。如下所示：

```c++
FormattedText ft("This is a brave new world");
ft.capitalize(10, 15);
cout << ft << endl;
// prints "This is a BRAVE new world"
```

但是，同样地，为每个字符定义一个布尔标志是非常不高效的，其实只需要起始和结束标记即可。让我们再次尝试使用享元（Flyweight）模式。

## Flyweight Implementation

让我们实现一个利用享元（Flyweight）设计模式的 `BetterFormattedText`。我们将首先定义外部类以及作为内部类的享元（因为为什么不呢？）：

```c++
class BetterFormattedText
{
public:
  struct TextRange
  {
    int start, end;
    bool capitalize;
    // other options here, e.g. bold, italic, etc.

    bool covers(int position) const
    {
      return position >= start && position <= end;
    }
  };

private:
  string plain_text;
  vector<TextRange> formatting;
};
```

如您所见，`TextRange` 只存储它所应用的起始和结束位置，以及实际的格式化信息——即我们是否想要大写文本以及其他任何格式选项（如粗体、斜体等）。它只有一个成员函数 `covers()`，帮助我们确定这个格式化设置是否需要应用于给定位置的字符。

`BetterFormattedText` 存储了一个 `TextRange` 享元的 `vector`，并且能够根据需要构造新的 `TextRange`：

```c++
TextRange& get_range(int start, int end)
{
  formatting.emplace_back(TextRange{ start, end });
  return *formatting.rbegin();
}
```

在前面的例子中有三件事情发生：

1. 构建了一个新的 `TextRange`。
2. 它被移动到了 `vector` 中。
3. 返回了最后一个元素的引用。

在前述的实现中，我们实际上并没有检查重复的范围——这本来也是符合享元（Flyweight）模式节省空间的精神的。

现在我们可以为 `BetterFormattedText` 实现 `operator<<`：

```c++
friend std::ostream& operator<<(std::ostream& os,
  const BetterFormattedText& obj)
{
  string s;
  for (size_t i = 0; i < obj.plain_text.length(); i++)
  {
    auto c = obj.plain_text[i];
    for (const auto& rng : obj.formatting)
    {
      if (rng.covers(i) && rng.capitalize)
        c = toupper(c);
      s += c;
    }
  }
  return os << s;
}
```

同样地，我们所做的只是遍历每个字符并检查是否有任何范围覆盖它。如果有，我们就应用该范围所指定的格式，在我们的情况下，就是大写化。请注意，这种设置允许范围自由重叠。

现在我们可以使用我们构建的所有内容来大写化那个相同的单词，尽管使用的是一个稍微不同且更灵活的 API：

```c++
BetterFormattedText bft("This is a brave new world");
bft.get_range(10, 15).capitalize = true;
cout << bft << endl;
// prints "This is a BRAVE new world"
```

## 总结

享元（Flyweight）模式从根本上说是一种节省空间的技术。它的具体实现形式多种多样：有时，享元作为 API 令牌返回，允许你修改创建它的对象；而在其他时候，享元是隐式的，隐藏在幕后——就像在我们 `User` 的例子中，客户端并不需要知道实际上使用了享元。
