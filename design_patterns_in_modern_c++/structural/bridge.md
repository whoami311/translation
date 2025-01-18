# Bridge

如果您一直关注C++编译器（特别是GCC、Clang和MSVC）的最新进展，您可能已经注意到编译速度正在提升。特别是，编译器变得越来越增量化，因此编译器实际上可以只重建那些发生变化的定义，而重用其余部分，而不是重建整个翻译单元。

我提到 C++ 编译的原因是，过去开发者们一直使用一种“奇特的技巧”来尝试优化编译速度。

当然，我指的是……

## The Pimpl Idiom

让我首先解释一下 Pimpl 惯用法（Pimpl idiom）的技术细节。假设您决定创建一个 `Person` 类，用于存储一个人的名字并允许他们打印问候语。而不是像平常那样定义 `Person` 的成员，您会这样定义这个类：

```c++
struct Person
{
  std::string name;
  void greet();

  Person();
  ~Person();

  class PersonImpl;
  PersonImpl* impl; // good place for gsl::owner<T>
};
```

这确实有点奇怪。对于一个简单的类来说，似乎工作量很大。我们来看看……我们有名字和 `greet()` 函数，但为什么需要费心去写构造函数和析构函数呢？还有这个 `PersonImpl` 类是怎么回事？

您看到的是一个选择将其实现隐藏在另一个类中的类，这个辅助类被恰当地命名为 `PersonImpl`。需要注意的关键点是，这个类不是在头文件中定义的，而是位于 .cpp 文件中（例如 Person.cpp），因此 `Person` 和 `PersonImpl` 是在同一个地方定义的。它的定义非常简单：

```c++
struct Person::PersonImpl
{
  void greet(Person* p);
}
```

原始的 `Person` 类提前声明了 `PersonImpl`，并继续保持一个指向它的指针。正是这个指针在 `Person` 的构造函数中被初始化，并在析构函数中被销毁；如果您觉得更好，完全可以使用智能指针。

```c++
Person::Person()
  : impl(new PersonImpl) {}

Person::~Person() { delete impl; }
```

现在，我们来实现 `Person::greet()` 方法，正如您可能已经猜到的，它只是将控制权传递给 `PersonImpl::greet()`：

```c++
void Person::greet()
{
  impl->greet(this);
}

void Person::PersonImpl::greet(Person* p)
{
  printf("hello %s", p->name.c_str());
}
```

所以……这就是 Pimpl 惯用法的核心，唯一的问题是“为什么？！”为什么要费尽周折，委托 `greet()` 方法并传递 `this` 指针呢？这种方法有三个优势：

- 更多的实现细节实际上被隐藏了。如果您的 `Person` 类需要一个丰富的 API，包含许多 `private` / `protected` 的成员，您会暴露所有这些细节给客户端，即使他们由于 `private` / `protected` 访问修饰符而无法访问那些成员。使用 Pimpl 后，他们只能获得 `public` 接口。
- 修改隐藏的 `Impl` 类的数据成员不会影响二进制兼容性。
- 头文件只需要包含声明所需的头文件，而不是实现所需的。例如，如果 `Person` 需要一个类型为 `vector<string>` 的 `private` 成员，您将被迫在 `Person.h` 头文件中 `#include <vector>` 和 `<string>`（这是传递性的，因此任何使用 Person.h 的人都会包括它们）。使用 Pimpl 惯用法，这可以在 .cpp 文件中完成。

您会注意到上述几点使我们能够保持一个干净、不变的头文件。这种做法的一个副作用是减少了编译时间。而且，对我们来说重要的是，Pimpl 实际上是桥接模式的一个很好的说明：在这种情况下，pimpl *opaque pointer*（opaque 与 transparent 相对，即您不知道它背后是什么）充当了桥梁，将 `public` 接口的成员与其隐藏在 .cpp 文件中的底层实现连接起来。

## Bridge

Pimpl 惯用法是桥接设计模式的一个非常具体的示例，所以让我们来看一些更通用的内容。假设我们有两种类（在数学意义上）的对象：几何形状和可以在屏幕上绘制它们的渲染器。

就像我们对适配器模式的说明一样，我们将假设渲染可以以矢量和光栅形式发生（尽管我们不会在这里编写任何实际的绘图代码），并且就形状而言，我们仅限于圆形。

首先，这里是 `Renderer` 基类：

```c++
struct Renderer
{
  virtual void render_circle(float x, float y, float radius) = 0;
};
```

我们可以很容易地构建矢量和光栅实现；我将仅通过一些代码来模拟实际渲染，这些代码会将内容写入控制台：

```c++
struct VectorRenderer : Renderer
{
  void render_circle(float x, float y, float radius) override
  {
    cout << "Rasterizing circle of radius " << radius << endl;
  }
};

struct RasterRenderer : Renderer
{
  void render_circle(float x, float y, float radius) override
  {
    cout << "Drawing a vector circle of radius " << radius << endl;
  }
};
```

基类 `Shape` 将保持对渲染器的引用；形状将支持通过成员函数 `draw()` 进行自我渲染，并且还将支持 `resize()` 操作：

```c++
struct Shape
{
protected:
  Renderer& renderer;
  Shape(Renderer& renderer) : renderer{ renderer } {}
public:
  virtual void draw() = 0;
  virtual void resize(float factor) = 0;
};
```

您会注意到 `Shape` 类对 `Renderer` 做了一个引用。这恰好是我们构建的桥梁。我们现在可以创建 `Shape` 类的一个实现，并提供额外的信息，比如圆心的位置以及半径。

```c++
struct Circle : Shape
{
  float x, y, radius;

  void draw() override
  {
    renderer.render_circle(x, y, radius);
  }

  void resize(float factor) override
  {
    radius *= factor;
  }

  Circle(Renderer& renderer, float x, float y, float radius)
    : Shape{renderer}, x{x}, y{y}, radius{radius} {}
};
```

好的，所以这种模式很快就展现出来了，而有趣的部分当然在于 `draw()` 方法：这就是我们使用桥梁将 `Circle`（它有关于其位置和大小的信息）连接到渲染过程的地方。而这里充当桥梁的确切事物是一个 `Renderer`，例如：

```c++
RasterRenderer rr;
Circle raster_circle{ rr, 5,5,5 };
raster_circle.draw();
raster_circle.resize(2);
raster_circle.draw();
```

在前述情况下，桥梁是 `RasterRenderer`：您创建它，并将引用传递给 `Circle`，从那时起，对 `draw()` 的调用将会使用这个 `RasterRenderer` 作为桥梁来绘制圆。如果您需要微调圆，可以调用 `resize()` 方法，渲染仍然会正常工作，因为渲染器并不知道也不关心 `Circle`，甚至不需要将其作为引用！

## Summary

桥接模式是一个相当简单的概念，充当连接器或粘合剂，将两部分连接在一起。通过使用抽象（接口），使得组件能够在不了解具体实现的情况下相互交互。

尽管如此，桥接模式的参与者确实需要意识到彼此的存在。具体来说，`Circle` 需要持有对 `Renderer` 的引用，而反过来，`Renderer` 知道如何专门绘制圆形（因此有名为 `draw_circle()` 的成员函数）。这与中介者模式形成对比，中介者模式允许对象在不直接知晓彼此的情况下进行通信。
