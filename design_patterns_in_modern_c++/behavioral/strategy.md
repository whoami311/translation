# Strategy

假设你决定将一个字符串数组或 `vector` 作为列表输出，就像这样：

- just
- like
- this

当你思考不同的输出格式时，你可能会意识到你需要遍历每个元素，并以某种额外的标记输出它们。但是在像 HTML 或 LaTeX 这样的语言中，列表还需要起始和结束标签或标记。在这些格式中处理列表的方式既相似（你需要输出每个项），又不同（项的输出方式）。每种格式都可以通过单独的策略来处理。

我们可以为渲染列表制定一个策略：

- 渲染起始标签 / 元素。
- 对于每个列表项，渲染该项。
- 渲染结束标签/元素。

对于不同的输出格式，可以制定不同的策略，并且这些策略可以被输入到一个通用的、不变的算法中以生成文本。

这是一种存在于动态（运行时可替换）和静态（模板组合，固定的）两种形式中的模式。让我们来看一下这两种形式。

## Dynamic Strategy

因此，我们的目标是以以下格式打印一个简单的文本项列表：

```c++
enum class OutputFormat
{
  markdown,
  html
};
```

我们策略的骨架将在以下基类中定义：

```c++
struct ListStrategy
{
  virtual void start(ostringstream& oss) {};
  virtual void add_list_item(ostringstream& oss, const string& item) {};
  virtual void end(ostringstream& oss) {};
};
```

现在让我们跳到我们的文本处理组件。这个组件将有一个特定于列表的成员函数，我们称之为 `append_list()`。

```c++
struct TextProcessor
{
  void append_list(const vector<string> items)
  {
    list_strategy->start(oss);
    for (auto& item : items)
      list_strategy->add_list_item(oss, item);
    list_strategy->end(oss);
  }

private:
  ostringstream oss;
  unique_ptr<ListStrategy> list_strategy;
};
```

所以我们有一个名为 `oss` 的缓冲区，所有输出都流向这里，我们用于渲染列表的策略，当然还有 `append_list()`，它指定了使用给定策略实际渲染列表所需采取的一系列步骤。

现在，请注意这里。组合，如前所述，是允许具体实现骨架算法的两个可能选项之一。

相反，我们可以添加像 `add_list_item()` 这样的虚拟成员函数，由派生类覆盖：这就是模板方法模式所做的。

无论如何，回到我们的讨论。我们现在可以继续实现不同的列表策略，例如 `HtmlListStrategy`：

```c++
struct HtmlListStrategy : ListStrategy
{
  void start(ostringstream& oss) override
  {
    oss << "<ul>\n";
  }
  void end(ostringstream& oss) override
  {
    oss << "</ul>\n";
  }
  void add_list_item(ostringstream& oss, const string& item) override
  {
    oss << "<li>" << item << "</li>\n";
  }
};
```

通过实现这些重写方法，我们填补了指定如何处理列表的空白。我们会以类似的方式实现 `MarkdownListStrategy`，但由于 Markdown 不需要起始 / 结束标签，我们只需要重写 `add_list_item()` 函数：

```c++
struct MarkdownListStrategy : ListStrategy
{
  void add_list_item(ostringstream& oss,
                     const string& item) override
  {
    oss << " * " << item << endl;
  }
};
```

我们现在可以开始使用 `TextProcessor`，为其提供不同的策略并获得不同的结果。例如：

```c++
TextProcessor tp;
tp.set_output_format(OutputFormat::markdown);
tp.append_list({"foo", "bar", "baz"});
cout << tp.str() << endl;

// Output:
// * foo
// * bar
// * baz
```

我们可以为策略在运行时可切换做出规定——这正是我们将这种实现称为动态策略的原因。这是通过 `set_output_format()` 函数完成的，其实现非常简单：

```c++
void set_output_format(const OutputFormat format)
{
  switch(format)
  {
    case OutputFormat::markdown:
      list_strategy = make_unique<MarkdownListStrategy>();
      break;
    case OutputFormat::html:
      list_strategy = make_unique<HtmlListStrategy>();
      break;
  }
}
```

现在，从一个策略切换到另一个非常简单，并且你可以立即看到结果：

```c++
tp.clear(); // clears the buffer
tp.set_output_format(OutputFormat::Html);
tp.append_list({"foo", "bar", "baz"});
cout << tp.str() << endl;

// Output:
// <ul>
//   <li>foo</li>
//   <li>bar</li>
//   <li>baz</li>
// </ul>
```

## Static Strategy

得益于模板的魔力，你可以将任何策略直接编译进类型中。对 `TextStrategy` 类只需要进行最小的更改：

```c++
template <typename LS>
struct TextProcessor
{
  void append_list(const vector<string> items)
  {
    list_strategy.start(oss);
    for (auto& item : items)
      list_strategy.add_list_item(oss, item);
    list_strategy.end(oss);
  }
  // other functions unchanged
private:
  ostringstream oss;
  LS list_strategy;
};
```

在前述内容中，我们所做的只是添加了 `LS` 模板参数，创建了一个该类型的成员策略，并开始使用它而不是之前使用的指针。`append_list()` 的结果是相同的：

```c++
// markdown
TextProcessor<MarkdownListStrategy> tpm;
tpm.append_list({"foo", "bar", "baz"});
cout << tpm.str() << endl;

// html
TextProcessor<HtmlListStrategy> tph;
tph.append_list({"foo", "bar", "baz"});
cout << tph.str() << endl;
```

前述示例的输出与动态策略的输出相同。请注意，我们必须创建两个 `TextProcessor` 实例，每个实例都有各自不同的列表处理策略。

## 总结

策略设计模式允许你定义一个算法的骨架，然后使用组合来提供与特定策略相关的缺失实现细节。这种方法存在两种形式：

- 动态策略仅仅保持对所使用策略的指针 / 引用。想要更换不同的策略吗？只需更改引用即可。非常简单！
- 静态策略要求你在编译时选择策略并坚持使用——之后没有改变主意的余地。

应该使用动态还是静态策略呢？嗯，动态策略允许你在对象构造后重新配置它们。想象一下控制文本输出格式的 UI 设置：你更愿意拥有一个可切换的 `TextProcessor` 还是两个类型分别为 `TextProcessor<MarkdownStrategy>` 和 `TextProcessor<HtmlStrategy>` 的变量？这完全取决于你。

最后，你可以限制类型可以接受的策略集：不是允许一个通用的 `ListStrategy` 参数，而是可以采用一个列出唯一允许传递类型的 `std::variant`。
