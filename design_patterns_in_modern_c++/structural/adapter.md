# Adapter

我以前经常旅行，一个让我可以把欧洲插头插入英式或美式插座的旅行转换器是一个非常好的比喻，可以用来解释适配器模式（Adapter pattern）的工作原理：我们被提供了一个接口，但我们想要的是另一个不同的接口，而通过构建一个适配器来包装原有接口，正是让我们达到目的的方法。

## Scenario

这里有一个简单的例子：假设您正在使用一个非常擅长绘制像素的库。而您自己则处理几何对象——如线、矩形等。您希望继续使用这些对象，但同时也需要该库提供的渲染功能，因此您需要将您的几何对象适配为基于像素的表示。

让我们从定义这个例子中（相当简单的）领域对象开始：

```c++
struct Point
{
  int x, y;
};

struct Line
{
  Point start, end;
};
```

现在让我们来讨论向量几何。一个典型的向量对象可能由一系列 `Line` 对象定义。与其继承自 `vector<Line>`，我们只需定义一对纯虚迭代器方法：

```c++
struct VectorObject
{
  virtual std::vector<Line>::iterator begin() = 0;
  virtual std::vector<Line>::iterator end() = 0;
};
```

因此，通过这种方式，如果您想要定义一个矩形（Rectangle），您可以在一个类型为 `vector<Line>` 的字段中保存一系列线条，并简单地暴露其端点：

```c++
struct VectorRectangle : VectorObject
{
  VectorRectangle(int x, int y, int width, int height)
  {
    lines.emplace_back(Line{ Point{x, y}, Point{x + width, y} });
    lines.emplace_back(Line{ Point{x + width, y}, Point{x + width, y + hegiht} });
    lines.emplace_back(Line{ Point{x, y}, Point{x, y + hegiht} });
    lines.emplace_back(Line{ Point{x, y + height}, Point{x + width, y + hegiht} });
  }

  std::vector<Line>::iterator begin() override {
    return lines.begin();
  }

  std::vector<Line>::iterator end() override {
    return lines.end();
  }
private:
  std::vector<Line> lines;
};
```

现在，这里是设定。假设我们想要在屏幕上绘制线条。甚至是矩形！不幸的是，我们不能直接做到，因为绘制的唯一接口实际上是这样的：

```C++
void DrawPoints(CPaintDC& dc, std::vector<Point>::iterator start, std::vector<Point>::iterator end)
{
  for (auto i = start; i != end; ++i)
    dc.SetPixel(i->x, i->y, 0);
}
```

我在这里使用的是 MFC（Microsoft Foundation Classes）中的 `CPaintDC` 类，但这不是重点。重点是我们需要像素。而我们只有线条。我们需要一个适配器。

## Adapter

好的，那么假设我们想要绘制几个矩形：

```c++
vector<shared_ptr<VectorObject>> vectorObjects{
  make_shared<VectorRectangle>(10, 10, 100, 100),
  make_shared<VectorRectandle>(30, 30, 60, 60)
}
```

为了绘制这些对象，我们需要将每一个从一系列的线条转换成相当数量的点。为此，我们创建一个单独的类来存储这些点，并以一对迭代器的形式提供访问。

```c++
struct LineToPointAdapter
{
  typedef vector<Point> Points;

  LineToPointAdapter(Line& line)
  {
    // TODO
  }

  virtual Points::iterator begin() { return points.begin(); }
  virtual Points::iterator end() { return points.end(); }
private:
  Points points;
};
```

转换在线条到多个点的构造函数中即时发生，因此适配器是急切的。实际的转换代码也相当简单：

```c++
LineToPointAdapter(Line& line)
{
  int left = min(line.start.x, line.end.x);
  int right = max(line.start.x, line.end.x);
  int top = min(line.start.y, line.end.y);
  int bottom = max(line.start.y, line.end.y);
  int dx = right - left;
  int dy = line.end.y - line.start.y;

  // only vertical or horizontal lines
  if (dx == 0)
  {
    // vertical
    for (int y = top; y <= bottom; ++y)
    {
      points.emplace_back(Point{ left, y });
    }
  }
  else if (dy == 0)
  {
    for (int x = left; x <= right; ++x)
    {
      points.emplace_back(Point{ x, top });
    }
  }
}
```

上述代码非常简单：我们只处理完全垂直或水平的线条，而忽略所有其他情况。我们现在可以使用这个适配器来实际渲染一些对象。我们取自示例中的两个矩形，并像这样简单地渲染它们：

```c++
for (auto& obj : vectorObjects)
{
  for (auto& line : *obj)
  {
    LineToPointAdapter lpo{ line };
    DrawPoints(dc, lpo.begin(), lpo.end());
  }
}
```

太棒了！我们所做的就是，对于每个向量对象，获取它的每一条线，为该线构造一个 `LineToPointAdapter`，然后迭代由适配器生成的点集，并将这些点传递给 `DrawPoints()`。而且它确实有效！（相信我，它是可以工作的。）

## Adapter Temporaries

然而，我们的代码存在一个主要问题：`DrawPoints()` 会在每次屏幕刷新时被调用，这意味着对于相同的线条对象，适配器会重复生成相同的数据，可能多达无数次。对此我们能做些什么呢？

一方面，我们可以在应用程序启动时预先定义所有点，例如：

```c++
vector<Point> points;
for (auto& o : vectorObjects)
{
  for (auto& l : *o)
  {
    LineToPointAdapter lpo{ l };
    for (auto& p : lpo)
      points.push_back(p);
  }
}
```

然后，`DrawPoints()` 的实现简化为：

```c++
DrawPoints(dc, points.begin(), points.end());
```

但是，让我们假设一下，原始的 `vectorObjects` 集合可能会发生变化。在这种情况下，缓存那些点是没有意义的，但我们仍然希望避免持续不断地重新生成可能重复的数据。我们如何处理这个问题呢？当然是通过缓存！

首先，为了避免重新生成数据，我们需要有唯一的方式去识别线条，这间接地意味着我们需要有唯一的方式去识别点。可以使用类似 ReSharper 的“生成 | 哈希函数（Generate | Hash function）”来帮助实现这一目标：

```c++
struct Point
{
  int x, y;

  friend std::size_t hash_value(const Point& obj)
  {
    std::size_t seed = 0x725C686F;
    boost::hash_combine(seed, obj.x);
    boost::hash_combine(seed, obj.y);
    return seed;
  }
};

struct Line
{
  Point start, end;

  friend std::size_t hash_value(const Line& obj)
  {
    std::size_t seed = 0x719E6B16;
    boost::hash_combine(seed, obj.start);
    boost::hash_combine(seed, obj.end);
    return seed;
  }
};
```

在前面的例子中，我选择了 Boost 的哈希实现。现在，我们可以构建一个新的  `LineToPointCachingAdapter`，使得它能够缓存点，并且只在必要时重新生成它们。实现几乎相同，只是有以下细微差别。

首先，适配器现在有了一个缓存：

```c++
static map<size_t, Points> cache;
```

这里的类型 `size_t` 正是 Boost 的哈希函数返回的类型。现在，当涉及到迭代生成的点时，我们如下方式产出它们：

```c++
virtual Points::iterator begin() { return cache[line_hash].begin(); }
virtual Points::iterator end() { return cache[line_hash].end(); }
```

这是算法中有趣的部分：在生成点之前，我们检查这些点是否已经生成。如果已经生成，我们就直接退出；如果没有生成，我们就生成它们并将它们添加到缓存中：

```c++
LineToPointCachingAdapter(Line& line)
{
  static boost::hash<Line> hash;
  line_hash = hash(line); // note: line_hash is a field!
  if (cache.find(line_hash) != cache.end())
    return; // we already have it

  Points points;

  // same code as before

  cache[line_hash] = points;
}
```

太好了！得益于哈希函数和缓存，我们大幅减少了进行的转换数量。唯一剩下的问题是，在点不再需要之后如何移除旧的点。这个具有挑战性的问题就留给读者作为练习了。

## Summary

适配器是一个非常简单概念：它允许您将现有的接口适应为您所需的接口。适配器唯一真正的问题是，在适应过程中，有时您会生成临时数据以满足其他数据表示的需求。当这种情况发生时，转向缓存：确保只有在必要时才生成新数据。此外，如果您希望在缓存对象发生变化时清理过期数据，则需要做更多的工作。

我们还没有真正解决的另一个问题是懒加载（laziness）：当前的适配器实现在创建时就执行转换。如果只想在适配器实际被使用时才进行工作呢？这相对容易实现，并且留给读者作为练习。
