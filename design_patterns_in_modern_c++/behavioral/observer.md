# Observer

## Property Observers

```c++
struct Person
{
    int age;
    Person(int age) : age{age} {}
};
```

```c++
struct Person
{
    int get_age() const { return age; }
    void set_age(const int value) { age = value; }
private:
    int age;
};
```

## Observer\<T\>

```c++
struct PersonListener
{
    virtual void person_changed(Person& p, const string& property_name, const any new_value) = 0;
};
```

```c++
template<typename T>
struct Observer
{
    virtual void field_changed(T& source, const string& field_name) = 0;
};
```

```c++
struct ConsolePersonObserver : Observer<Person>
{
    void field_changed(Person& source, const string& field_name) override
    {
        cout << "Person's " << field_name << " has changed to " << source.get_age() << ".\n";
    }
};
```

```c++
struct ConsolePersonObserver : Observer<Person>, Observer<Creature>
{
    void field_changed(Person& source, ...) { ... };
    void field_changed(Creature& source, ...) { ... };
}
```

## Observable\<T\>

```c++
template <typename T>
struct Observable
{
    void notify(T& source, const string& name) { ... }
    void subscribe(Observer<T>* f) { observers.push_back(f); }
    void unsubscribe(Observer<T>* f) { ... }
private:
    vector<Observer<T>*> observers;
};
```

```c++
void notify(T& source, const string& name)
{
    for (auto obs : observers)
        obs->field_changed(source, name);
}
```

```c++
struct Person : Observable<Person>
{
    void set_age(const int age)
    {
        if (this->age == age) return ;
        this->age = age;
        notify(*this, "age");
    }
private:
    int age;
};
```

## Connecting Observers and Observables

```c++
struct ConsolePersonObserver : Observer<Person>
{
    void field_changed(Person& source, const string& field_name) override
    {
        cout << "Person's " << field_name << " has changed to " << source.get_age() << ".\n";
    }
};
```

```c++
Person p{ 20 };
ConsolePersonObserver cpo;
p.suscribe(&cpo);
p.set_age(21); // Person's age has changed to 21.
p.set_age(22); // Person's age has changed to 22.
```

## Dependency Problems

```c++
bool get_can_vote() const { return age >= 16; }
```

```c++
void set_age(int value) const
{
    if (age == value) return;

    auto old_can_vote = can_vote(); // store old value
    age = value;
    notify(*this, "age");

    if (old_can_vote != can_vote()) // check value has changed
        notify(*this, "can_vote");
}
```

## Unsubscription and Thread Safety

```c++
void unsubscribe(Observer<T>* observer)
{
    observer.earse(remove(observers.begin(), observers.end(), observer), observers.end());
};
```

```c++
template <typename T>
struct Observable
{
    void notify(T& source, const string& name)
    {
        scope_lock<mutex> lock{ mtx };
        ...
    }
    void subscribe(Observer<T>* f)
    {
        scope_lock<mutex> lock{ mtx };
        ...
    }
    void unsubscribe(Observer<T>* o)
    {
        scope_lock<mutex> lock{ mtx };
        ...
    }
private:
    vector<Observer<T>*> observers;
    mutex mtx;
};
```

## Reentrancy

```c++
struct TrafficAdministration : Observer<Person>
{
    void TrafficAdministration::field_changed(Person& source, const string& filed_name) override
    {
        if (field_name == "age")
        {
            if (source.get_age() < 17)
                cout << "Whoa there, you are not old enough to drive!\n";
            else
            {
                // oh, ok, they are old enough, let's not monitor them anymore
                cout << "We no longer care!\n";
                source.unsubscribe(this);
            }
        }
    }
};
```

```c++
notify() --> field_changed() --> unsubscribe()
```

```c++
void unsubscribe(Observer<T>* o)
{
    auto it != find(observers.begin(), observers.end(), o);
    if (it != observers.end())
        *it = nullptr;  // cannot do this for a set
}
```

```c++
void notify(T& source, const string& name)
{
    for (auto obs : observers)
    {
        if (obs)
            obs->field_changed(source, name);
    }
}
```

```c++
void notify(T& source, const string& name)
{
    vector<Observer<T>*> observers_copy;
    {
        lock_guard<mutex_t> lock{ mtx };
        observers_copy = observers;
    }
    for (auto obs : observers_copy)
        if (obs)
            obs->field_changed(source, name);
}
```

## Observer via Boost.Signals2

```c++
template <typename T>
struct Observable
{
    signal<void(T&, const string&)> property_changed;
};
```

```c++
struct Person : Observable<Person>
{
    ...
    void set_age(const int age)
    {
        if (this->age == age) return;

        this->age = age;
        property_changed(*this, "age");
    }
};
```

```c++
Person p{123};
auto conn = p.property_changed.connect([](Person&, const string& prop_name)
{
    cout << prop_name << " has been changed" << endl;
});
p.set_age(20);  // name has been changed

// later, optionally
conn.disconnect();
```

## Summary
