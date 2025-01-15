# Singleton

## Singleton as Global Object

```c++
struct Database
{
    /**
    * \brief Please do not create more than one instance.
    */
    Database() {}
};
```

```c++
static Database database{};
```

```c++
Database& get_database()
{
    static Database database;
    return database;
}
```

## Classic Implementation

```c++
struct Database
{
    Database()
    {
        static int instance_count{ 0 };
        if (++instance_count > 1)
            throw std::exception("Cannot make >1 database!");
    }
};
```

```c++
struct Database
{
protected:
    Database() { /* do what you need to do */ }
public:
    static Database& get()
    {
        // thread-safe in C++11
        static Database database;
        return database;
    }
    Database(Database const&) = delete;
    Database(Database&&) = delete;
    Database& operator=(Database const&) = delete;
    Database& operator=(Database &&) = delete;
};
```

```c++
static Database& get() {
    static Database* database = new Database();
    return *database;
}
```

### Thread Safety

```c++
struct Database
{
    // same members as before, but then...
    static Database& instance();
private:
    static boost::atomic<Database*> instance;
    static boost::mutex mtx;
};

Database& Database::instance()
{
    Database* db = instance.load(boost::memory_order_consume);
    if (!db)
    {
        boost::mutex::scoped_lock lock(mtx);
        db = instance.load(boost::memory_order_consume);
        if (!db)
        {
            db = new Database();
            instance.store(db, boost::memory_order_release);
        }
    }
}
```

## The Trouble with Singleton

```c++
class Database
{
public:
    virtual int get_population(const std::string& name) = 0;
};
```



```c++
class SingletonDatabase : public Database
{
    SingletonDatabase() { /* read data from database */ }
    std::map<std::string, int> capitals;
public:
    SingletonDatabase(SingletonDatabase const&) = delete;
    void operator=(SingletonDatabase const&) = delete;

    static SingletonDatabase* get()
    {
        static SingletonDatabase db;
        return db;
    }

    int get_population(const std::string& name) override
    {
        return capitals[name];
    }
};
```

```c++
struct SingletonRecordFinder
{
    int total_population(std::vector<std::string> names)
    {
        int result = 0;
        for (auto& name : names)
            result += SingletonDatabase::get().get_population(name);
        return result;
    }
};
```

```c++
TEST(RecordFinderTests, SingletonTotalPopulationTest)
{
    SingletonRecordFinder rf;
    std::vector<std::string> names{ "Seoul", "Mexico City" };
    int tp = rf.total_population(names);
    EXPECT_EQ(17500000 + 17400000, tp);
}
```

```c++
struct ConfigurableRecordFinder
{
    explicit ConfigurableRecordFinder(Database& db)
        : db{db} {}

    int total_population(std::vector<std::string> names)
    {
        int result = 0;
        for (auto& name : names)
            result += db.get_population(name);
        return result;
    }

    Database& db;
}
```

```c++
class DummyDatabase : public Database
{
    std::map<std::string, int> capitals;
public:
    DummyDatabase()
    {
        capitals["alpha"] = 1;
        capitals["beta"] = 2;
        capitals["gamma"] = 3;
    }

    int get_population(const std::string& name) override {
        return capitals[name];
    }
};
```

```c++
TEST(RecordFinderTests, DummyTotalPopulationTest)
{
    DummyDatabase db{};
    ConfigurableRecordFinder rf{ db };
    EXPECT_EQ(4, rf.total_population(std::vector<std::string>{"alpha", "gamma"}));
}
```

## Singletons and Inversion of Control

```c++
auto injector = di::make_injector(
    di::bind<IFoo>.to<Foo>.in(di::singleton),
    // other configuration steps here
);
```

## Monostate

```c++
class Printer
{
    static int id;
public:
    int get_id() const { return id; }
    void set_id(int value) { id = value; }
};
```

## Summary
