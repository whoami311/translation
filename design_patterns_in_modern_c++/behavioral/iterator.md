# Iterator

每当您开始处理复杂的数据结构时，都会遇到遍历的问题。这可以通过不同的方式来处理，但最常见的遍历方式，比如遍历一个 `vector`，是使用一种称为迭代器（iterator）的对象。

迭代器，简单来说，是一个可以指向集合中元素的对象，并且知道如何移动到集合中的下一个元素。因此，它只需要实现 `++` 操作符和 `!=` 操作符（以便您可以比较两个迭代器并检查它们是否指向同一事物）。就是这样。

C++ 标准库大量使用了迭代器，所以我们将会讨论它们在标准库中的使用方式，然后我们将看一下如何创建我们自己的迭代器以及迭代器的局限性是什么。

##  Iterators in the Standard Library

想象您有一个如下所示的名字列表：

```c++
vector<string> names{ "john", "jane", "jill", "jack" };
```

如果您想要获取名字集合中的第一个名字，您会调用一个叫做 `begin()` 的函数。这个函数不会直接通过值或引用给您返回第一个名字；相反，它会给您一个迭代器：

```c++
vector<string>::iterator it = names.begin();
// begin(names)
```

函数 `begin()` 既作为 `vector` 的成员函数存在，也作为一个全局函数存在。全局版本对于数组（C 风格数组，而非 `std::array`）特别有用，因为它们不能有成员函数。

因此，`begin()` 返回一个您可以视为指针的迭代器：在 `vector` 的情况下，它具有类似的机制。例如，您可以取消引用（dereference）迭代器来打印实际的名字：

```c++
cout << "first name is " << *it << "\n";
// first name is john
```

我们获得的迭代器知道如何前进，也就是说，移动到指向下一个元素。重要的是要意识到，`++` 指的是向前移动的概念，这与指针的 `++` 不同，指针的 `++` 是在内存中向前移动（即，增加一个内存地址）：

```c++
++it; // now points to jane
```

我们也可以使用迭代器（与指针相同的方式）来修改它所指向的元素：

```c++
it->append(" goodall"s);
cout << "full name is " << *it << "\n";
// full name is jane goodall
```

现在，`begin()` 的对应函数当然是 `end()`，但它并不指向最后一个元素——相反，它指向最后一个元素之后的位置。这里有一个不太准确的图示：

```
        1 2 3 4
begin() ^       ^ end()
```

您可以使用 `end()` 作为终止条件。例如，让我们使用我们的迭代器变量 `it` 来打印剩余的名字：

```c++
while (++it != names.end())
{
  cout << "another name: " << *it << "\n";
}
// another name: jill
// another name: jack
```

除了 `begin()` 和 `end()` 之外，我们还有 `rbegin()` 和 `rend()`，它们允许我们反向遍历集合。在这种情况下，如您所猜测的，`rbegin()` 指向最后一个元素，而 `rend()` 指向第一个元素之前的位置：

```c++
for (auto ri = rbegin(names); ri != rend(names); ++ri)
{
  cout << *ri;
  if (ri + 1 != rend(names)) // iterator arithmetic
    cout << ", ";
}
cout << endl;
// jack, jill, jane goodall, john
```

在前述内容中有两点值得指出。首先，即使我们是反向遍历向量，我们仍然对迭代器使用 `++` 操作符。其次，我们可以进行算术运算：同样，当我写 `ri + 1` 时，这指的是 `ri` 之前的一个元素，而不是之后。

我们还可以拥有不允许修改对象的常量迭代器：它们通过 `cbegin()` 和 `cend()` 返回，并且当然也有反向版本 `crbegin()` 和 `crend()`：

```c++
vector<string>::const_reverse_iterator jack = crbegin(names);
// won't work
*jack += "reacher";
```

最后，值得一提的是现代 C++ 中的范围基于的 for 循环（range-based for loop），它作为从容器的 `begin()` 一直迭代到 `end()` 的简写形式：

```c++
for (auto& name : names)
  cout << "name = " << name << "\n";
```

请注意，这里的迭代器是自动解引用的：变量 `name` 是一个引用，但您也可以通过值来迭代。

## Traversing a Binary Tree

让我们来完成经典的“计算机科学”练习——遍历二叉树。首先，我们定义这个树的节点如下：

```c++
template <typename T>
struct Node
{     
  T value;  
  Node<T> *left = nullptr;
  Node<T> *right = nullptr;
  Node<T> *parent = nullptr;
  BinaryTree<T>* tree = nullptr;
};
```

每个节点都有指向其左子树和右子树的引用，如果有的话还有指向其父节点的引用，以及指向整个树的引用。节点可以独立构建，也可以在其子节点已经指定的情况下构建。

```c++
explicit Node(const T& value)
  : value(value)
{
}

Node(const T& value, Node<T>* const left, Node<T>* const right)
  : value(value),
    left(left),
    right(right)
{
  this->left->tree = this->right->tree = tree;
  this->left->parent = this->right->parent = this;
}
```

最后，我们引入一个实用的成员函数来设置 `tree` 指针。这个操作会递归地应用到节点的所有子节点上：

```c++
void set_tree(BinaryTree<T>* t)
{
  tree = t;
  if (left) left->set_tree(t);
  if (right) right->set_tree(t);
}
```

有了这些准备，我们现在可以定义一个叫做 `BinaryTree` 的结构——正是这个结构将允许我们进行遍历。

```c++
template <typename T>
struct BinaryTree
{
  Node<T>* root = nullptr;

  explicit BinaryTree(Node<T>* const root)
    : root{ root }
  {
    root->set_tree(this);
  }
};
```

现在我们可以为树定义一个迭代器。遍历二叉树有三种常见的方式，我们将首先实现的是前序遍历（preorder）：

- 我们在遇到元素时立即返回它。
- 我们递归地遍历左子树。
- 我们递归地遍历右子树。

因此，让我们从构造函数开始：

```c++
template <typename U>
struct PreOrderIterator
{
  Node<U>* current;

  explicit PreOrderIterator(Node<U>* current)
    : current(current)
  {
  }

  // other members here
};
```

我们需要定义 `operator !=` 来与其他迭代器进行比较。由于我们的迭代器行为类似于指针，因此这个操作非常简单：

```c++
bool operator!=(const PreOrderIterator<U>& other)
{
  return current != other.current;
}
```

我们还需要 `*` 操作符用于解引用：

```c++
Node<U>& operator*() { return *current; }
```

现在，最难的部分来了：遍历树。这里的挑战在于我们不能使算法递归——请记住，遍历发生在 `++` 操作符中，所以我们最终会这样实现它：

```c++
PreOrderIterator<U>& operator++()
{
  if (current->right)
  {
    current = current->right;
    while (current->left)
      current = current->left;
  }
  else
  {
    Node<T>* p = current->parent;
    while (p && current == p->right)
    {
      current = p;
      p = p->parent;
    }
    current = p;
  }
  return *this;
}
```

确实，这样的实现看起来相当混乱！而且，它与经典的树遍历实现相去甚远，原因正是我们没有使用递归。我们稍后再回到这个话题。

现在，最后的问题是如何从我们的 `BinaryTree` 暴露迭代器。如果我们定义它为树的默认迭代器，我们可以如下填充其成员：

```c++
typedef PreOrderIterator<T> iterator;

iterator begin()
{
  Node<T>* n = root;

  if (n)
    while (n->left)
      n = n->left;
  return iterator{ n };
}

iterator end()
{
  return iterator{ nullptr };
}
```

值得一提的是，在 `begin()` 中，迭代并不是从整个树的根节点开始；而是从最左边的可用节点开始。

现在所有部分都已经就绪，以下是我们将如何进行遍历：

```c++
BinaryTree<string> family{
  new Node<string>{"me",
    new Node<string>{"mother",
      new Node<string>{"mother's mother"},
      new Node<string>{"mother's father"}
    },
    new Node<string>{"father"}
  }
};

for (auto it = family.begin(); it != family.end(); ++it)
{
  cout << (*it).value << "\n";
}
```

您也可以将这种遍历方式暴露为一个独立的对象，即：

```c++
class pre_order_traversal
{
  BinaryTree<T>& tree;
public:
  pre_order_traversal(BinaryTree<T>& tree) : tree{tree} {}
  iterator begin() { return tree.begin(); }
  iterator end() { return tree.end(); }
} pre_order;
```

用于如下方式：

```c++
for (const auto& it: family.pre_order)
{
  cout << it.value << "\n";
}
```

同样地，可以定义中序（in_order）和后序（post_order）遍历算法来暴露适当的迭代器。

## Iteration with Coroutines

我们有一个严重的问题：在我们的遍历代码中，`operator++` 是一段难以阅读的混乱代码，与你在维基百科上读到的树遍历方法不匹配。它确实能工作，但它只能工作是因为我们将迭代器预先初始化为从最左边的节点开始，而不是从根节点开始，这同样可以说是存在问题且令人困惑的。

为什么我们会遇到这个问题？因为 `++` 操作符的函数是不可恢复的：它无法在调用之间保持其栈状态，因此递归是不可能的。现在，如果我们有一种机制可以让我们两全其美：既能保持函数状态又能执行正确的递归，那会怎样呢？这正是协程（coroutines）的作用所在。

使用协程，我们可以如下实现后序树遍历：

```c++
generator<Node<T>*> post_order_impl(Node<T>* node)
{
  if (node)
  {
    for (auto x : post_order_impl(node->left))
      co_yield x;
    for (auto y : post_order_impl(node->right))
      co_yield y;
      co_yield node;
  }
}

generator<Node<T>*> post_order()
{
  return post_order_impl(root);
}
```

这难道不棒吗？算法终于再次变得可读了！而且，这里完全看不到 `begin()` 和 `end()`：我们只是返回一个生成器（generator），这是一种专门设计用来逐步返回值的类型，这些值是通过 `co_yield` 提供给它的。在每个值被产出之后，我们可以暂停执行并做其他事情（比如说，打印这个值），然后在不丢失上下文的情况下恢复迭代。这使得递归成为可能，并允许我们编写如下代码：

```c++
for (auto it: family.post_order())
{
  cout << it->value << endl;
}
```

协程是 C++ 的未来，解决了许多传统迭代器要么显得笨拙要么不适合的问题。

## 总结

迭代器设计模式在 C++ 中以显式和隐式（例如，基于范围的形式）广泛存在。不同类型的迭代器适用于遍历不同的对象：例如，反向迭代器可能适用于 `vector`，但不适用于单链表。

实现您自己的迭代器只需要提供 `++` 和 `!=` 操作符。大多数迭代器只是指针的外观（façades），旨在一次性遍历集合后就被丢弃。

协程修复了迭代器中存在的一些问题：它们允许在调用之间保存状态——这是其他语言（如 C#）很早就已经实现的功能。因此，协程使我们能够编写递归算法。
