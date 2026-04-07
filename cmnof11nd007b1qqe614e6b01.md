---
title: "Hashing in C++26"
seoTitle: "Hashing in C++26"
seoDescription: "How to write generic hash function using C++26 reflection."
datePublished: 2026-04-07T09:27:15.099Z
cuid: cmnof11nd007b1qqe614e6b01
slug: hashing-in-cpp-26
tags: reflection, cpp26

---

In my previous post, I showed a generic approach to hashing (see https://blog.infotraining.pl/how-to-hash-objects-without-repetition). That implementation required a `tied()` member that exposes an object's members as an `std::tuple` of references. With such a `tied()` implementation, we could provide a specialization of `std::hash<T>` to compute an object's hash.

A type could opt in to this generic hashing in two ways:

*   inheriting from the `HashableForTiedMembers` tag struct,
    
*   or defining a `constexpr` template variable that enables the concept used by the `std::hash<T>` specialization.
    

This design works well in C++20 and C++23. With C++26 (and GCC 16), however, language-level reflection lets us implement the same functionality without `tied()`, enabling a simpler and more robust reflection-based solution.

## Introduction to C++ Reflection

C++26 introduces a long‑awaited feature that changes how we do metaprogramming. Reflection lets us inspect — and in some cases manipulate — program structure at compile time, simplifying tasks such as generic hashing, serialization, and code generation.

### Reflection operator

The reflection operator `^^` is used to obtain a compile-time representation of an entity. It can be applied to various constructs such as variables, functions, types, namespaces, and enumerators. The result of applying the reflection operator is a reflection-value that contains information about the entity, such as its members, member names, types, and attributes. This value can be stored in a variable of opaque type `std::meta::info` and can be queried at compile-time using various reflection queries - **meta-functions** (defined in `<meta>` header).

```cpp
struct Data { int id; std::string name };

std::meta::info r_type = ^^Data;
std::meta::info r_id_member = ^^Data::id;
```

Information obtained through reflection can be used for performing compile-time checks, generating code, or implementing custom behavior based on the structure of the code. The reflection value can be used to produce grammatical elements using so-called **splicers** with `[: reflection-value :]` syntax, this allows to generate code based on the structure of the code at compile time.

```cpp
Data data{42, "forty-two"};

assert(data.[: r_id_member :] == 42); // the same as: data.id == 42
```

### Simple example

Let's see a simple example that allows us to get member values using indexes:

```cpp
#include <meta>
#include <string>

struct Person {
    int id;
    std::string name;
};

consteval std::meta::info member_number(int n) {
  if (n == 0) 
    return ^^Person::id; 
  else if (n == 1) 
    return ^^Person::name; 

  std::unreachable();
}

int main() {
  Person p{42, "John Doe"};

  p.[: member_number(0) :] = 665;  // Same as: p.id = 42;
  p.[: member_number(1) :] = "John Smith"; // Same as: p.name = "John Smith";
  p.[: member_number(5) :] = "John Doe";   // Error: member_number(5) is not a constant
}
```

In the example above, we define a struct Person with two members: `id` and `name`. We then create a consteval function, `member_number`, which takes an integer `n` and returns the reflection of the corresponding member of `Person`. This enables us to associate indexes with the struct's members. The `consteval` keyword ensures the function is evaluated at compile time, as reflection values can only be used in compile-time contexts.

In the `main` function, we create an instance of `Person` and use the splicer syntax to assign values to the members based on their index.

There's no need to manually index struct items. We can utilize the standard library's meta-function `std::meta::nonstatic_data_members_of(^^T, context)`. This function accepts a reflected value of a type `^^Person`) and returns a `std::vector` of the type's reflected non-static data members, allowing easy iteration or indexing.

Simplified implementation of `member_number()` function can look like this:

```c++
using namespace std::meta;

consteval info member_number(int n) {
  auto ctx = access_context::current();
  return nonstatic_data_members_of(^^Person, ctx)[n];
}
```

Another important aspect is the use of `std::meta::access_context::current()` to obtain the current access context. This is crucial because the meta-function must determine which entities are visible and accessible at the reflection point. The access context provides details about the scope and visibility of entities, allowing us to accurately reflect on the members of `Person`. If we need to reflect on private members, we can use `std::meta::access_context::unchecked()`.

## Hashing with Reflection

After this brief introduction to C++26 reflection, let's explore how it can be used to implement hashing for our types. In C++, a hashable type is one for which a specialization of the `std::hash` template exists. This specialization specifies how to compute a hash value for an object of that type.

```cpp
template <>
struct std::hash<Person> {
    size_t operator()(const Person& p) const {
        size_t seed = 0;
        Utility::hash_combine(seed, p.id);   // Hashing an integer
        Utility::hash_combine(seed, p.name); // Hashing a string
        return seed;
    }
};
```

Before C++26, we had to either write our own specialization of `std::hash` for each type we wanted to hash or use a generic approach that employed `std::tie()` (or similar methods) to combine the hash values of individual members. I discussed the latter approach in my previous article. Now, we can implement a more elegant and efficient solution using reflection.

To begin, let's define a concept called `Hashable`. This concept will verify if a type is hashable by determining whether a specialization of `std::hash` exists for that type.

```cpp
template <typename T>
concept Hashable = requires {
    { std::hash<T>{}(std::declval<T>()) } -> std::convertible_to<size_t>;
};

static_assert(Hashable<int>); // int is hashable
static_assert(Hashable<std::string>); // std::string is hashable
static_assert(Hashable<Person>); // Person is hashable
```

Next, we can create a generic hash function that utilizes reflection to compute the hash value for any hashable type. This function will iterate over the non-static data members of the type and combine their hash values using `Utility::hash_combine()` or a similar function like `boost::hash_combine()`.

```cpp
template <typename T>
    requires std::is_class_v<T>
size_t calculate_hash(const T &obj, size_t seed = 0)
{
    constexpr auto ctx = std::meta::access_context::unchecked();
    
    static constexpr auto r_data_members = 
        std::define_static_array(
            std::meta::nonstatic_data_members_of( ^^T, ctx)
        );

    // iteration over non-static data members
    template for (constexpr auto r_dm : r_data_members)
    {
        const auto& member_value = obj.[: r_dm :];
        Utility::hash_combine(seed, member_value);
    }

    return seed;
}
```

In this implementation, we first verify if the type `T` is a class or struct using `std::is_class_v<T>` trait. We then retrieve the non-static data members of the type with `std::meta::nonstatic_data_members_of( ^^T, ctx)`, where `ctx` serves as the access context. To iterate over public, protected, and private members, we use `std::meta::access_context::unchecked()`, which grants access to all members regardless of their access specifier. Since `std::meta::nonstatic_data_members_of` returns a `std::vector`, we define a static array, `r_data_members`, to store these reflected data members. This setup allows us to employ a template for loop to iterate over the members at compile time. The standard library's `std::define_static_array()` helps bridge compile-time and runtime by defining a static array from a compile-time sequence. Using a template for loop, we iterate over the reflected data members. For each member, we access its value for the given object obj using the splicer syntax `obj.[: r_dm :]`, and then combine the hash value.

### Hashing Base Classes

Now, we can compute the combined hash value for all non-static data members. However, there's an important aspect we must address: the base classes or structs also need to be included in the hash calculation. Thanks to reflection, this task is less daunting than it might appear.

For a given type `T`, we can query its base classes using the meta-function `std::meta::bases_of(^^T, ctx)`. After defining a static array of these reflected base types, we can iterate over them. To obtain the reflected value of a type, we apply `std::meta::type_of(r_base)`. The entire snippet appears as follows:

```cpp
static constexpr auto r_base_types = std::define_static_array(std::meta::bases_of(^^T, ctx));
```

```cpp
static constexpr auto r_base_types = 
  std::define_static_array(std::meta::bases_of(^^T, ctx));

template for (constexpr auto r_base : r_base_types)
{
  using Base = typename[:std::meta::type_of(r_base):];
  static_assert(Hashing::Hashable<Base>, "Base class must be hashable");
  Utility::hash_combine(seed, static_cast<const Base &>(obj));
}
```

Once again we can use splicer to define an alias for a base type. When we want to obtain a type from refected value of type we need to use `typename` before.

Now the whole template function looks like this:

```cpp
template <typename T>
    requires std::is_class_v<T>
size_t calculate_hash(const T &obj, size_t seed = 0)
{
    constexpr auto ctx = std::meta::access_context::unchecked();

    // combine hashes of base classes
    static constexpr auto r_base_types = std::define_static_array(std::meta::bases_of(^^T, ctx));

    template for (constexpr auto r_base : r_base_types)
    {
        using Base = typename[:std::meta::type_of(r_base):];
        static_assert(Hashing::Hashable<Base>, "Base class must be hashable");
        Utility::hash_combine(seed, static_cast<const Base &>(obj));
    }

    // combine hashes of non-static data members
    static constexpr auto r_data_members = std::define_static_array(std::meta::nonstatic_data_members_of(^^T, ctx));

    template for (constexpr auto r_dm : r_data_members)
    {
        const auto& member_value = obj.[:r_dm:];
        Utility::hash_combine(seed, member_value);
    }

    return seed;
}
```

## Enabling Hashing for Custom Types

Now it is almost complete. The final consideration is how to enable specialization of `std::hash` for a custom type. One approach is to optin to our hashing implementation by defining an alias, `enabled_for_hashing`, within a custom class. This alias activates the generation of a hash value using our implementation.

```cpp
struct EnabledForHashing_t {};

template <typename T>
concept EnabledForHashing = requires {
   typename T::enabled_for_hashing;
};

// specialization for types opted-in for hashing
template <EnabledForHashing T>
struct std::hash<T>
{
    size_t operator()(const T& obj) const
    {
        return calculate_hash(obj);
    }
};

class Person
{
    using enabled_for_hashing = EnabledForHashing_t;

    //...
};
```

## **Conclusion**

C++26 reflection lets us replace the boilerplate and fragility of a `tied()` - based hashing approach with a cleaner, more robust solution. By using the reflection operator (`^^`) to inspect an object's structure at compile time, we can generate consistent, maintainable hash implementations without requiring every type to expose a `std::tuple` of members or to opt in via tags or template variables. The result is less repetitive code, fewer user-facing APIs to learn, and a lower risk of forgetting a member when you extend a type.

That said, adoption comes with practical considerations. Reflection-based hashing improves ergonomics and enables more work at compile time (including `constexpr` hashing) but relies on compiler support and the current state of language/library tooling. For projects that must support older standards or compilers, keeping a fallback (the `tied()` pattern or explicit `std::hash` specializations) is reasonable. For new code targeting modern toolchains (e.g., GCC 16 and other compilers that implement C++26 reflection), the reflection approach is worth adopting.

Going forward, use reflection-based hashing to reduce maintenance burden, but: test hash behaviour for stability across builds if you depend on hash persistence, document any members you intentionally skip from hashing, and pay attention to compiler bugs and implementation details as toolchains mature. Overall, C++26 reflection is a strong step toward safer, more concise metaprogramming—and hashing is a clear, practical win.

As a next step, we can extend our implementation with features like attributes that will allow us to discard some values for calculating the hash, but I think it is a nice subject for a next post.

The example of code can be found at: https://godbolt.org/z/1covqd36r