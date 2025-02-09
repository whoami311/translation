# Facade

首先，让我们解决语言上的问题：字母 Ç 中的那个小曲线叫做软音符号（cedilla），这个字母本身发音为/S/，因此单词‘façade’发音为 /fɑːˈsɑːd/。对于特别讲究的读者，欢迎在代码中使用字母 ç，因为大多数编译器都能很好地处理它。

好了，现在谈谈那个模式……

我在量化金融和算法交易领域工作了很长时间。可以想象，一个好的交易终端需要快速将信息传递到交易员的大脑：您希望所有事情尽可能快地呈现，没有任何延迟。

大部分金融数据（除了图表）实际上是用纯文本渲染的：黑色屏幕上的白色字符。这在某种程度上类似于您操作系统中的终端 / 控制台 / 命令行界面的工作方式，但有一个细微的区别。

## How the Terminal Works

终端窗口的第一部分是缓冲区（buffer）。这是存储已渲染字符的地方。缓冲区是内存中的一个矩形区域，通常是 1D 或 2D 的 `char` 或 `wchar_t` 数组。缓冲区可以比终端窗口的可见区域大得多，因此它可以存储一些您可以回滚查看的历史输出。

通常，缓冲区有一个指针（例如，一个整数）来指定当前输入行。这样，当缓冲区满时不会重新分配所有行；它只是覆盖最旧的那一行。

然后是视口（viewport）的概念。视口渲染缓冲区的一部分。由于缓冲区可能非常大，所以视口只取出缓冲区中的一个矩形区域并渲染该部分。显然，视口的大小必须小于或等于缓冲区的大小。

最后是控制台（终端窗口）本身。控制台显示视口，允许上下滚动，并且甚至接受用户输入。事实上，控制台是一个外观（façade）：它是对幕后相当复杂设置的简化表示。

通常，大多数用户与单个缓冲区和视口进行交互。然而，也可以拥有一个将区域在两个视口之间垂直分割的控制台窗口，每个视口都有其对应的缓冲区。这可以通过诸如 Linux 命令 `screen` 之类的工具来实现。

## An Advanced Terminal

典型操作系统终端的一个问题是，如果您向其管道输入大量数据，它会变得非常慢。例如，Windows 终端窗口（cmd.exe）使用 GDI 来渲染字符，这是完全不必要的。在快速交易环境中，您希望渲染是硬件加速的：字符应该作为预渲染纹理通过如 OpenGL 这样的 API 放置在一个表面上。

一个交易终端由多个缓冲区和视口组成。在典型的设置中，不同的缓冲区可能会并发地从各个交易所或交易机器人接收更新的数据，所有这些信息都需要显示在单个屏幕上。

缓冲区还提供了远比简单的 1D 或 2D 线性存储更令人兴奋的功能。例如，`TableBuffer` 可以被定义为：

```c++
struct TableBuffer : IBuffer
{
  TableBuffer(vector<TableColumnSpec> spec, int totalHeight) { ... }

  struct TableColumnSpec
  {
    string header;
    int width;
    enum class TableColumnAlignment {
      Left, Center, Right
    } alignment;
  };
};
```

换句话说，一个缓冲区（buffer）可以接受一些规范并构建一个表格（是的，一个古老的传统ASCII格式化的表格！），并将其呈现在屏幕上。

视口（viewport）负责从缓冲区获取数据。它的一些特性包括：

- 正在显示的缓冲区的引用
- 它的尺寸
- 如果视口比缓冲区小，它需要指定将要显示缓冲区的哪一部分。这通过绝对 x-y 坐标来表达。
- 在整个控制台窗口中的视口位置
- 光标的定位，假设此视口当前正在接收用户输入

## Where’s the Façade?

控制台本身在这个特定的系统中充当外观（façade）。内部而言，控制台必须管理大量不同的对象：

```c++
struct Console
{
  vector<Viewport*> viewports;
  Size charSize, gridSize;
  ...
};
```

控制台的初始化通常也是一个非常复杂的过程。然而，由于它是作为一个外观（Façade），它实际上试图提供一个非常易用的 API。这可能需要若干合理的参数来初始化所有内部组件。

```c++
Console::Console(bool fullscreen, int char_width, int char_height,
  int width, int height, optional<Size> client_size)
{
  // single buffer and viewport created here
  // linked together and added to appropriate collections
  // image textures generated
  // grid size calculated depending on whether we want fullscreen mode
}
```

或者，可以将所有这些参数封装到一个单独的对象中，该对象同样会有一些合理的默认值：

```c++
Console::Console(const ConsoleCreationParameters& ccp) { ... }

struct ConsoleCreationParameters
{
  optional<Size> client_size;
  int character_width{10};
  int character_height{14};
  int width{20};
  int height{30};
  bool fullscreen{false};
  bool create_default_view_and_buffer{true};
};
```

## 总结

外观（Façade）设计模式是一种在一個或多个复杂子系统前放置一个简单接口的方法。在我们的例子中，一个涉及许多缓冲区和视口的复杂设置可以直接使用；或者，如果您只是想要一个带有单个缓冲区和相关视口的简单控制台，您可以通过一个非常易用且直观的 API 来获得它。
