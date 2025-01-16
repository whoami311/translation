# Adapter

## Scenario

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

```c++
struct VectorObject
{
    virtual std::vector<Line>::iterator begin() = 0;
    virtual std::vector<Line>::iterator end() = 0;
};
```

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

```C++
void DrawPoints(CPaintDC& dc, std::vector<Point>::iterator start, std::vector<Point>::iterator end)
{
    for (auto i = start; i != end; ++i)
        dc.SetPixel(i->x, i->y, 0);
}
```

## Adapter

```c++
vector<shared_ptr<VectorObject>> vectorObjects{
    make_shared<VectorRectangle>(10, 10, 100, 100),
    make_shared<VectorRectandle>(30, 30, 60, 60)
}
```

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

```c++
for (auto& obj : vectorObjects)
{
    for (auto& line : *obj)
    {
        LineToPointAdapter lpo{ line };
        DeawPoints(dc, lpo.begin(), lpo.end());
    }
}
```

## Adapter Temporaries

```c++
vector<Point> points;
for (auto& o : vectorObjects)
{
    for (auto& l : *o)
    {
        LineToPointAdapter lpo{ l };
        for (auto& P : lpo)
            points.push_back(p);
    }
}
```

```c++
DrawPoints(dc, points.begin(), points.end());
```

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

```c++
static map<size_t, Points> cache;
```

```c++
virtual Points::iterator begin() { return cache[line_hash].begin(); }
virtual Points::iterator end() { return cache[line_hash].end(); }
```

这就是算法的有趣之处：在生成点之前，我们要检查它们是否已经生成。如果已经生成，我们就退出；如果还没有，我们就生成它们并添加到缓存中：

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

## Summary

