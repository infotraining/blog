---
title: "How to Hash Objects Without Repetition: std::hash can be DRY"
seoTitle: "Object Hashing Without Repetition"
seoDescription: "Learn to implement hash functions for classes without redundancy using std::hash and std::tie. Use fold expressions with std::tie and std::apply."
datePublished: 2024-12-06T12:56:48.134Z
cuid: cm4cr2oee000208l7dgagbini
slug: how-to-hash-objects-without-repetition
tags: cpp, concepts, tuples, hashing-functions

---

In my previous [article on `std::tie`](https://blog.infotraining.pl/comparisons-with-ties), I discussed how to simplify comparison operators for your classes using the `std::tie()` function.

Now we’ll explore a more complex case. Let's create a generic implementation of a hashing function for a class or structure we develop.

# Repetitive hashing

To provide a hash value for our custom object, we should write a `std::hash` specialization for our type.

A typical code example looks like this:

```cpp
struct Person
{
    std::string first_name;
    std::string last_name;
    std::uint8_t age;
};

template <>
struct std::hash<Person> 
{
    size_t operator()(const Person& p) const
    {
        std::size_t seed{};
        some_lib::hash_combine(seed, p.first_name);
        some_lib::hash_combine(seed, p.last_name);
        some_lib::hash_combine(seed, p.age);
        return seed;    
    }
};
```

To calculate the hash value, we need to call a function that incrementally combines the hash values of all our data members. This function often comes from an external library, like Boost - `boost::hash_combine()`. This task is repetitive and tedious. Let's find a better solution.

# Fold expressions & hash\_combine

We can start by writing a variadic version of `hash_combine()`. Such a function should accept any number of arguments and return combined hash for all of them. This can be done using a fold expression with the “famous” `,` operator:

```cpp
template <typename... TValues>
auto combined_hash(const TValues&... values)
{
    size_t seed{};
    (..., some_lib::hash_combine(seed, values));

    return seed;
}
```

Now we can simplify hashing with the following code:

```cpp
template <>
struct std::hash<Person> 
{
    size_t operator()(const Person& p) const
    {
        return combined_hash(p.first_name, p.last_name, p.age);
    }
};
```

It's now possible to pass data members as arguments to our function.

# Hashing & `std::tie`

As you might recall from my [previous post](https://blog.infotraining.pl/comparisons-with-ties), `std::tie()` can create a view that refers to the data members of our class. We used this to simplify the implementation of comparison operators:

```cpp
struct Person
{
    std::string first_name;
    std::string last_name;
    std::uint8_t age;

    auto tied() const
    {
        return std::tie(first_name, last_name, age);
    }

    bool operator==(const Person& other) const
    {
        return tied() == other.tied();
    }
};
```

Can we use the same trick in our hash implementation?

We can try to tie the data members together and combine their hash values like this:

```cpp
template <>
struct std::hash<Person> 
{
    size_t operator()(const Person& p) const
    {
        return combined_hash(p.tied());
    }
};
```

Now, we are passing a tuple as an argument to our `combined_hash()` function. Unfortunately, `std::tuple` does not have a `std::hash<>` specialization, so the compiler will generate an error. However, if we use `boost::hash_combine()`, it works because Boost can hash the tuples for us.

# std::apply to the rescue

Let's pause for a moment and reexamine the code.

Now, we can calculate a combined hash for any set of hashable values:

```cpp
std::string first_name = "John";
std::string last_name = "Smith";
uint8_t age = 42;
size_t hashed_value = combined_hash(first_name, last_name, age);
```

We need to call this function for arguments that are tied in a tuple. The code should look like this:

```cpp
std::tuple tpl{"John"s, "Smith"s, 42};

size_t hashed_value = combined_hash(std::get<0>(tpl), std::get<1>(tpl), std::get<2>(tpl));
```

It occurs that in the standard library we have `std::apply(F&& f, Tuple&& t)` function that allows calling any function `f` with elements of `t` passed as arguments.

```cpp
void foobar(const std::string& foo, int bar) 
{
    std::cout << "foobar(" << foo << ", " << bar << ")\n";
}

std::tuple args{"Foo"s, 42};
std::apply(foobar, args); // tha same as foobar(std::get<0>(args), std::get<1>(args))
```

Let’s apply `std::apply` to our needs :)

```cpp
template <>
struct std::hash<Person> 
{
    size_t operator()(const Person& p) const
    {
        return std::apply(combined_hash, p.tied());
    }
};
```

Unfortunately, this is not enough. The compiler will complain that it cannot deduce the template parameter `F` for `std::apply(F&& f, Tuple&& t)`. This is because our `combined_hash()` is a template function. We can easily solve this problem by wrapping the function in a closure (also known as a lambda). The working code will be as follows:

```cpp
template <>
struct std::hash<Person> 
{
    size_t operator()(const Person& p) const
    {
        auto hasher = [](const auto&... args) { return combined_hash(args...); };
        return std::apply(hasher, p.tied());
    }
};
```

# Generic hashing solution for future classes

This solution might seem complex, but it offers us a benefit. We can create a generic specialization for any custom type that has a `tied()` member function. We can provide a `std::hash<T>` specialization for each type that chooses to use our hashing implementation.

```cpp
struct HashableForTiedMembers
{ };

template <typename T>
constexpr bool hashing_for_tied_members = false; 

template <typename T>
concept HashableForTied = std::derived_from<T, HashableForTiedMembers> || hashing_for_tied_members<T>;

template <HashableForTied T>
struct std::hash<T>
{
    size_t operator()(const T& value) const
    {
        auto hasher = [](const auto&... args) {
            return combined_hash(args...);
        };
        return std::apply(hasher, value.tied());
    }
};
```

The creator of the class can choose from two opt-in options:

* A class can inherit from the `HashableForTiedMembers` struct, which serves as a tag, enabling the compiler to specialize `std::hash` for that type. However, this option requires our class or struct to have constructors. For aggregates that derive from empty-base classes, additional `{}` are needed in the argument list, changing the syntax of object initialization.
    
    ```cpp
    struct HashableForTiedMembers
    { };
    
    struct Person : HashableForTiedMembers
    {
        std::string first_name;
        std::string last_name;
        std::uint8_t age;
    
        Person(std::string first_name, std::string last_name, std::uint8_t age)
            : first_name{std::move(first_name)}
            , last_name{std::move(last_name)}
            , age{age}
        {
        }
    
        auto tied() const
        {
            return std::tie(first_name, last_name, age);
        }
    };
    ```
    
* Alternatively, we can define the opt-in tag externally using template variables.
    
    ```cpp
    struct AggregatePerson
    {
        std::string first_name;
        std::string last_name;
        std::uint8_t age;
    
        auto tied() const
        {
            return std::tie(first_name, last_name, age);
        }
    };
    
    // opt-in hashing for AggregatePerson
    template <>
    constexpr bool hashing_for_tied_members<AggregatePerson> = true;
    ```
    

By using the `tied()` member function for hashing, we can easily implement `operator==`, which is needed for storing objects as keys in unordered associative containers. The following class can be used as a key in an `unordered_map`:

```cpp
class Coordinate : public HashableForTiedMembers
{
public:
    Coordinate(int x, int y) noexcept
        : x_{x}
        , y_{y}
    { }

    auto tied() const noexcept
    {
        return std::tie(x_, y_);
    }

    bool operator==(const Coordinate& other) const noexcept
    {
        return tied() == other.tied();
    }

private:
    int x_;
    int y_;
};

int main()
{
    std::unordered_map<Coordinate, std::string> coordinates;
	coordinates.emplace(Coordinate{0, 0}, "origin");
	coordinates.emplace(Coordinate{1, 0}, "right");
	coordinates.emplace(Coordinate{0, 1}, "up");
	coordinates.emplace(Coordinate{1, 1}, "up-right");
}
```

# Summary

In this blog post, I showed a way to implement hashing for types in a generic way.

I hope you find it interesting. Let me know if you do.

The sample code can be found at [https://godbolt.org/z/7767MP1zK](https://godbolt.org/z/7767MP1zK).