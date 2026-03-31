---
title: "How to Simplify Object Comparisons with Ties in C++11/14"
seoTitle: "Simplify Object Comparisons in C++11/14"
seoDescription: "Learn how to simplify object comparisons using `std::tie()` in C++11/14, reducing boilerplate code for equality and comparison operators"
datePublished: 2024-11-07T13:11:50.310Z
cuid: cm37bub6u000b09l61gm2ctvp
slug: comparisons-with-ties
tags: cpp, comparison, tuples

---

When we want to check the equality of objects or compare them, we need to provide appropriate comparison operators for our class. The implementation usually needs to delegate comparisons to the corresponding member variables.

The typical code might look like this:

```cpp
#include <string>
#include <cassert>
#include <cstdint>

struct Person
{
    std::string first_name;
    std::string last_name;
    std::uint8_t age;
    
    bool operator==(const Person& other) const
    {
        return first_name == other.first_name 
               && last_name == other.last_name
               && age == other.age;
    }

    bool operator<(const Person& other) const
    {
        if (first_name == other.first_name)
        {
            if (last_name == other.last_name)
            {
                return age < other.age;
            }
            
            return last_name < other.last_name;
        }

        return first_name < other.last_name;
    }
};

int main()
{
    Person p1{"John", "Doe", 33};
    Person p2{"John", "Doe", 33};
    Person p3{"John", "Don", 44};

    assert(p1 == p2);
    assert(p2 < p3);
}
```

We can simplify such a mundane task (especially for `operator <`) when we use `std::tie()` function from C++11. The code can be rewritten to:

```cpp
#include <string>
#include <tuple>
#include <cassert>
#include <cstdint>

struct Person
{
    std::string first_name;
    std::string last_name;
    std::uint8_t age;
    
    bool operator==(const Person& other) const
    {
        return std::tie(first_name, last_name, age) == std::tie(other.first_name, other.last_name, other.age);
    }

    bool operator<(const Person& other) const
    {
        return std::tie(first_name, last_name, age) < std::tie(other.first_name, other.last_name, other.age);
    }
};
```

In C++14, we can make things even simpler by reducing repetition.

```cpp
struct Person
{
    std::string first_name;
    std::string last_name;
    uint8_t age;

    auto tied() const
    {
        return std::tie(first_name, last_name, age);
    }

    bool operator==(const Person& other) const
    {
        return tied() == other.tied();
    }

    bool operator<(const Person& other) const
    {
        return tied() < other.tied();
    }
};
```

# How it works?

## Tuples with references

Let's start with what are known as reference tuples. These are tuples that hold references to specific values.

```cpp
std::string name = "John";
std::uint8_t age = 42;

std::tuple<std::string&, std::uint8_t&> ref_tpl(name, age);
```

The created tuple called `ref_tpl` holds references to two variables: `name` and `age`. We can access them using the `std::get<Index>()` function:

```cpp
std::cout << std::get<0>(ref_tpl) << "\n"; // prints: "John"

std::get<1>(ref_tpl) = 33; // assigns 33 to age variable
```

The tuple named `ref_tpl` holds references to two variables: `name` and `age`. We can access these variables using the `std::get<Index>()` function.

```cpp
ref_tpl = std::make_tuple("Adam", 24);
assert(name == "Adam");
assert(age == 33);
```

## std::tie

Now `std::tie()` comes in handy. We can easily create reference tuples like this:

```cpp
auto ref_tpl = std::tie(name, age); // returns: std::tuple<std::string&, std::uint8_t&>
```

This function determines the types of the arguments and returns a tuple that holds references to the passed lvalue arguments.

So the `tied()` method in our struct returns tuple that hold references to object’s members:

```cpp
struct Person
{
    std::string first_name;
    std::string last_name;
    uint8_t age;

    auto tied() const
    {
        return std::tie(first_name, last_name, age);
    }
};
```

## Comparing tuples

Tuples can be compared in lexicographic order. They perform comparisons based on their corresponding members.

```cpp
std::tuple<std::string, std::uint8_t> tpl1("John", 33);
std::tuple<std::string, std::uint8_t> tpl2("John", 33);

assert(tpl1 == tpl2);
assert(tpl1 == std::make_tuple("John", 33));
assert(tpl1 < std::make_tuple("John", 44));
```

You can also compare reference tuples:

```cpp
std::string name = "John";
std::uint8_t age = 42;

assert((std::tie(name, age) == std::tuple<std::string, std::uint8_t>("John", 42)));
assert(std::tie(name, age) < std::make_tuple("John", 45));
```

## Putting It All Together

Now our simplified code should be easy to understand.

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

    bool operator<(const Person& other) const
    {
        return tied() < other.tied();
    }
};
```

The `tied()` member function returns a tuple with references to the object's members in a specific order. This tuple works like a read-only view of the object's members and can be compared with another tuple, letting us write any comparison operator in a single line.