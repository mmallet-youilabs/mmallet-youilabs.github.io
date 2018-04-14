---
title:  "Understanding CYIAny"
author: mathieu_mallet
---
`CYIAny` is a You.i Engine One class that is poorly understood. While some people love its flexibility, others dislike the problems it cause while debugging. This post attemps to explain how `CYIAny` works, why it works this way, and how to better work with `CYIAny` instances.

# Description

`CYIAny` is a type that performs type erasure in order to support storying 'any' type into itself.

Why would someone want to use this? In an ideal world, the type of object that needs to be stored in a container is known ahead of time. For example:

```c++
class Car
{
    Engine m_engine;
    std::vector<Wheel> m_wheels;
};
```

In this case, we know that a `Car` always has an engine, and some amount of wheels. Different types of engines would be dealt with by subclassing from the `Engine` class (or interface). The same would be done for the different types of wheels. This works because we can enforce a common base class for engines and wheels. But what if we had a more generic class?

```c++
class Cabinet
{
    std::map<CYIString, File> m_files; // key is the file's tag
};
```

Here, we want to create a `Cabinet` class that can contain multiple `File` objects. Each object has a specific name. This works because all each item in the cabinet is of a type that is known ahead of time. Now what if we wanted an even more general inventory container?

```c++
class Closet
{
    std::map<CYIString, Item> m_items; // key is the item's tag
};
```

This works, but requires that all items stored in the closet inherit from `Item` (or implement the `Item` interface). What if we wanted to put wheels in the closet? Now the `Wheel` class has to subclass the `Item` class (or implement the `Item` interface), which would pollute the the `Wheel` class. What we really need is a way to store any type within the closet, without needing to specify the type ahead of time in the closet class. Enter `CYIAny`:

```c++
class Closet
{
    std::map<CYIString, CYIAny> m_items; // key is the item's tag
};
```

Now, users can store any type of item in the `Closet` type, and the `Closet` type does not need to know what those types are. Users that take items _out_ of the `Closet`, however, will need to know what the type is.

This is the use of `CYIAny`: it allows classes to be written without needing to know what the types that it will contain ahead of time.

Note that, if at all possible, `CYIAny` should not be used: instead, classes should be written where the stored type is known. This can be done by storing the type directly in the class, or by making the class use a templated type. This makes the code easier to read, easier to debug, and more efficient. But in those (few) cases where the type cannot be known (e.g. because you're writing a library that takes user data as input), then consider using `CYIAny`.

# Implementation

At a minimum, an 'any' class needs two things:
- Some way to identify what is stored within it
- Storage space for the data

Such a class may look something like this:
```c++
class Any
{
public:
    template<typename T>
    void Set(T value);

    template<typename T>
    T Get() const;

private:
    std::reference_wrapper<const std::type_info> m_type;
    void *m_pData;
};
```

The `Set` function would assign create a copy of the `value` parameter and assign it to `m_pData`, and assign the type of the value to `m_type`. To get the value out, a user would call `Get()` and provite a type (e.g. `Get<CYIString>()`). The `Any` class would check if the provided type matches the contained type, and if so return a copy of the contained type.

This would be one way to do it, but this way has a major drawback: every value assigned to the `Any` class would result in a heap allocation, even if the stored type is small. This is both wasteful and slow: more memory is needed to store the data, and heap allocations are somewhat slow (compared to stack allocations).

This is why most 'any' classes implement 'small buffer optimization'. For types that are small enough, the data for the type is stored directly inside the `Any` object rather than performing a heap allocation. This is generally done by allocating a block of memory, and either doing placement new within that block (for small types) or storing a pointer to a heap-allocated object in that block (for large types). Some implementation use an union between a block of memory and a void* pointer for legibility.

(Side note: `std::string` is another class that is often implemented with 'small buffer optimization': memory is set aside within `std::string` to store small strings. This avoids relatively expensive heap allocation, at the cost of possibly using more memory.)

The `CYIAny` class actually looks like this:

```c++
class CYIAny
{
public:
    template<typename T>
    void Set(T value);

    template<typename T>
    T Get() const;

private:
    FunctionsTable *m_pFunctions;
    void *m_pStorage[2];
};
```

(In the real implementation, more functions exist, the names are different and the setter is actually `operator=`... but this is roughly accurate.)

When an object is assigned to a `CYIAny` instance, the type of the object is used to locate and assign a function table to the `m_pFunctions` variable. The function table contains functions to assign an object into `m_pStorage`, to retrieve an object from `m_pStorage`, and to destroy the value contained in `m_pStorage`. These functions are templated on the type of object being assigned/accessed/deleted, and therefore know how to deal with the contained type. These functions are also templated on a flag that indicate if the object is 'small', in which case the object is stored inline in `m_pStorage`, or if the object is 'large', in which case the object is stored in the heap (and a pointer to the data is stored in `m_pStorage`).

Note that the previously shown 'Any' implementation wouldn't work for complex types (e.g. types with a non-trivial destructor), as that Any implementation has no way to know which destructor to call.

# std::any

The C++17 standard adds an 'any' implementation as `std::any`. While the specific implementation is left to the compiler, the typical implementation also uses a functions table and a block of pre-allocated memory.

The std::any implementation lives under the `<any>` header. Since C++17 was only recently ratified, some compilers may still be using `<experimental\any>`.

# Debugging

`CYIAny` implements debugging-time type safety. If the `CYIAny::Get<T>()` function is called with a type other than the contained type, an assert is triggered. When necessary, the type contained in `CYIAny` can be checked using `CYIAny::ContainsType<T>()`.

One major drawback of 'any' classes is that their content is opaque both to the compiler and to the user. This means that the content of `CYIAny` cannot be viewed when instances of that class are examined in a debugger:

![CYIAny viewed in debugger](/assets/img/cyiany/cyiany_in_debugger.png)

By digging into the type table, we can tell that this `CYIAny` contains a `CYIString` object, but we can't see what the string value is.

There are two ways that we can see what the actual `CYIAny` object contains:
1. By using the `CYIAny::ToString()` function
2. By using an union to show possible types in the debugger

## Debugging using CYIAny::ToString()

Let's examine the first method. `CYIAny::ToString()` doesn't have to be called from code -- it can also be called from the debugger. Using Xcode/LLDB as example:

```tcl
(lldb) print exampleAny.ToString()
(CYIString) $1 = (m_string = "\"Hello world!\"", m_uLength = 0)
(lldb)
```

In Xcode, the calling of `ToString()` can be automated using the following [summary expression](https://youilabs.atlassian.net/wiki/spaces/EN/pages/24674329/XCode#XCode-SummaryFormat):

```{$VAR.ToString().GetData()}:s```

When viewed in Xcode, the `exampleAny` variable then becomes:

![CYIAny viewed in debugger with summary expression](/assets/img/cyiany/cyiany_in_debugger_with_summary_expression.png)

`CYIAny::ToString()` depends on the type being 'printable' using `operator<<` and uses `CYIObjectPrinter` to do the conversion to string. Support for custom types can be added by implementing the following function:

```std::ostream &operator<<(std::ostream &stream, const MyCustomType &val);```

## Debugging using unions

This method requires changes to be made to the `YiAny.h` file, which isn't practical for users of You.i Engine One. However, the upcoming 5.0 version of You.i Engine One is expected to support debugging this way 'out of the box' (see [US-7692](https://youilabs.atlassian.net/browse/US-7692)).

When debugging with an union, an union is created and contains the preallocated `CYIAny` storage. Typical/expected types are added to this union, which allows the debugger to show the type values. For example:

![CYIAny viewed in debugger with union](/assets/img/cyiany/cyiany_in_debugger_with_union.png)

The user, however, needs to know what the contained type is in order to know which entry to look at. This can be done either with an extra 'type name' field, or by relying on the function types in the functions table. The 'type name' field can be populated with the result of `typeid(value).name()`, though the result of that function call is compiler-specific (and not always human-readable). In the previous example, the type is shown to be '9CYIString' (meaning `CYIString`), and the content can be examined under the `m_pStringValue` entry.

There are a few other drawbacks to this debugging method:
- The size of types can differ between platforms, which makes it difficult to predict if a type would be allocated within the storage or in the heap. For example, `std::string` has a different size between gcc and clang (since clang makes use of small buffer optimization).
- Types in union must be trivially-constructible. This means that if a non-trivially-constructible type is small enough to be allocated within the `CYIAny`'s storage, it cannot be viewed as part of the union.
- Storing the type name increases the size of a `CYIAny` instance by ~8 bytes. This can be mitigated by only allocating that field (as well as the union types) when building in Debug.

# Drawbacks

The difficult to debug is likely the biggest drawback of `CYIAny`, but there are a few others:
- `CYIAny` was implemented before C++11 support was added to You.i Engine One. As a result, it has poor (or no) support for move operations.
- `CYIAny` cannot contain instances of itself. Interestingly, this drawback is not present in `std::any`.

# What about variants?

A variant class is a type that can store one or more other types. It differs from 'any' classes in that the list of types it can store is pre-defined, either in the class itself or as template types of the of class.

The C++17 standard adds a 'variant' implementation as `std::variant`.

You.i Engine One also provides a variant class, `CYIVariant`, though its supported types are fixed rather than user-specifiable. It is used exclusively within the animation subsystem to hold animatable values. Internally, `CYIVariant` uses `CYIAny` to hold the supported types, though this isn't strictly necessary (and could be accomplished with a simple union and type enum).

When possible, 'variant' classes should be used in lieu of 'any' classes.

# Performance

Using a `CYIAny` type has some performance overhead:
- Accesses must be through a function table
- Large types must be allocated on the heap

For types small enough to be allocated within the `CYIAny` storage itself, using `CYIAny` is nearly as fast as using a type instance directly. However, large types need to be allocated on th heap, which can have a significant overhead.

Current versions of You.i Engine One allocate 8 bytes of storage within `CYIAny`. Types larger than 8 bytes are allocated on the heap, while types 8 bytes or smaller are stored directly in `CYIAny`'s storage. This proved to be too low: during animations, `glm::vec3` objects are stored in a `CYIVariant`, which internally uses a `CYIAny` object. As a result, small objects are continually allocated on the heap when playing position or scale animations.

Started in version 5.0.0 of You.i Engine One, the internal storage of `CYIAny` is being increased to 16 bytes. This allows `glm::vec3` objects to be stored in `CYIAny` without any heap allocations, and significantly reduces the number of heap allocations made when animations are played. Note that this can result in a higher memory usage when only very small types (e.g. 8 bytes maximum) are being stored in `CYIAny` objects. See [US-8106](https://youilabs.atlassian.net/browse/US-8106) for details.

# Uses in You.i Engine One

The `CYIAny` class is used internally in a few locations in You.i Engine One:
- As part of the `CYIVariant` implementation
- In `CYIBundle` to implement a key-value map that can hold any value type
- In `CYIAbstractDataModel` to store any value types
