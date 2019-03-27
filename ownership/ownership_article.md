# Ownership

## What I mean by "ownership"

Ownership is about lifetimes of objects and resources and who needs to manage them.

Ownership is a term that's common in the "modern" C++ community, though the concepts it denotes are more broadly applicable.

Ownership is the responsibility of making sure that an object is created when it is needed, maintained while it is still necessary, and cleaned up exactly once when it is no longer needed, including freeing any other resources or whatever else it might need to do when destroyed.

I'm not sure if the term exists in other contexts, but ownership is a concept that exists outside of modern C++, it exists even when using older style C++ or C or even other languages; it is a fundamental part of a program having resources that it must manage.

The kinds of issues that can come from not thinking about ownership are:

* memory leaks
* multiple deallocation of the same memory
* deallocation with the wrong deleter/free function (`delete` vs `delete[]`)
* attempting to refer to objects that no longer exist or never did
* forgetting to release (non-memory) resources when you don't need them any more

## RAII

Resource Acquisition Is Initialisation

Asking for a place to put a thing actually gives you the thing.

This means you cannot have uninitialised objects in C++.

This is funny, because every C++ developer knows it isn't true, but it is true:
In C++, built-in types declared with *automatic storage* and no explicit initialisation are initialised to indeterminate values, which is in practice the same thing as the values being uninitialised, but the standard calls this "default initialization".

Class or struct types will have their constructor called, which will in turn initialise all its data members (recursively), using default initialisation unless explicitly initialised.

### Ridiculous Acronym Isn't Intuitive

RAII **also** refers to the fact that the object will be cleaned up as soon as it goes out of scope - the destructor will be called and the memory will be deallocated.

This feature of C++, combined with destructors and *move semantics*, makes it possible to have compiler-enforced representation of particular types of ownership in C++.

This is why, as far as I know, "ownership" is most commonly talked about in the "modern" C++ community.

### Other languages and Garbage Collection

Other languages usually rely on a Garbage Collector to clean up objects which are no longer referenced by anything.

To C++ developers, garbage collectors have a bad reputation, but garbage collectors in general have advanced quite a long way and some are now quite good at not locking up the program while they run and not being susceptible to having reference cycles causing memory leaks.

However, garbage collected languages often don't provide a mechanism that's equivalent to C++'s RAII concept, where objects are guaranteed to be cleaned up in a deterministic and customisable way as soon as their scope is exited.

These other languages also tend not to have support for a destructor equivalent, which means managing resources that are not just memory allocations becomes tedious and potentially error prone.

Some garbage collected languages do have things to ensure objects are cleaned up at the end of a given function or scope,  like `defer` (Go) or `with` (Python).

Rust has a "borrow checker" which sounds pretty swell but I don't know Rust so `¯\_(ツ)_/¯`. However, I think this keeps track of ownership from more of a thread-safety perspective than an more general ownership tracking perspective.

I believe Rust does have an RAII like mechanism and move semantics.

#### Garbage Collection and C++

I know third party garbage collectors exist for C++, but I don't know what they are, how hard they are to use, or what their characteristics are.

As far as I know, there is no serious and widely supported intention to have Garbage Collection added to standard C++.

It would be swell to have a language that has good standard support for both objects managed by a standard garbage collector as well as objects managed by more explicit ownership modelling types like  smart pointers and RAII mechanisms like C++ has, as long as any given object is owned by either only the garbage collector or only the smart pointer.

## Smart pointers

Smart pointers are types which contain a pointer to an object, but are themselves an object following RAII principles, and which model ownership of the pointed to object.

Upon destruction of the smart pointer, the pointed-to object will be cleaned up if it is the last (compatible) smart pointer pointing to the object.

The clean up will involve calling the destructor of the pointed-to object, as well as deallocating the memory.

## Unique Ownership

Unique ownership is a model of resource management where there is a single point of ownership for an object.

This is the most common situation: the object exists somewhere for a predictable period, and anything that needs to refer to the object can do so at a predictable time within the object's lifetime, and can therefore safely do so without owning it.

Unfortunately, C++ doesn't really offer a way to have non-owning reference semantics to a uniquely owned object in such a way that the compiler can check that the referred-to object is still valid at the time of access, though static analysers are often able to detect this.

As such, knowing whether using non-owning reference semantics to uniquely owned objects is safe is up to the programmer.

There are some fairly easy ways to make sure of this safety though:

* have objects on the stack of a parent scope and pass references around
* have a long-lived object own all the collections of objects, like a game engine owning its objects - no access to an object should occur outside the lifetime of the engine
* make sure all threads with callbacks that refer to specific objects are joined before their owner is cleaned up

It is preferable not to rely on static lifetime as a way to make sure accesses are safe as this basically has all the downsides of a global variable, and it doesn't completely solve the lifetime issues:

* static clean up order is between hard and impossible to predict and debug
* still have to make sure references to an object don't survive longer than the module that the static variable is in

Examples of modelling unique ownership in C++ include

* Automatic storage variables (local variables, non-static data members, etc)
* C++ collections (vectors uniquely own the objects they contain, string uniquely owns its buffer, etc)
* `std::unique_ptr`

## Shared Ownership

Shared ownership is a model of resource management where there is multiple references to a single resource, and it is not possible to know at compile time which reference will be the last.

As such, the references themselves need to be tracked, and the resource being referred to should only be cleaned up when the last reference is clean up.

If it is possible to know at compile time which reference will be cleaned up last, it is more efficient to use a form of unique ownership, though there might be some rare situations where the ownership and lifetimes might be clearer with a type that models shared ownership and the ideally small performance impact might be worth avoiding bugs during maintenance in future.

An example of a situations where shared ownership is necessary or the best choice might be if you have multiple threads operating on a file, and you want to free the file when all the threads are finished, but you don't know at compile time which thread will take the longest.

`std::shared_ptr` is the only type in standard C++ which models shared ownership, although third party and in-house equivalents also exist and are reasonably common, and there are many ways to model shared ownership outside of using this particular type.

Unlike with `stdd::unique_ptr`, it *is* possible in C++ to have non-owning references to objects which are owned by `std::shared_ptr`s and ensure all access to that object are safe and only within the lifetime of the referred-to object; `std::weak_ptr` was made for this purpose.

It is not possible to move ownership of an object from a `std::shared_ptr` to a `std::unique_ptr`, because in the general case, there is no safe way to ensure that there is only one `std::shared_ptr` pointing to an object.

## Move Semantics

Move semantics is a term which refers to the concept of an object being passed from one location to another, and allowing C++ to optimise the operation.

This is necessary for performance in C++ because it is so common to use value types in C++, and objects are often non-trivial to construct, copy and destroy. It is common enough that one instance is no longer needed and another instance of the same value is needed elsewhere and as such C++11 standard introduced move semantics.

If an object owns memory that exists elsewhere, a move operation can simply pass ownership of that memory to the new object, whereas a deep copy would require another allocation and copying the data.

Copy-on-write is another solution to a similar set of problems, but it is not quite the same, and isn't specifically supported by the C++ standard.

The addition of move semantics allowed C++ to implement `std::unique_ptr` as a replacement for the broken `std::auto_ptr`; now the compiler could enforce that a `std::unique_ptr` could never be copied and that all passing of a `std::unqiue_ptr` would result in the old copy being nulled out.

Move semantics are also available to `std::shared_ptr` so that ownership can be moved without incrementing and decrementing the atomic reference counter, while also ensuring that the old pointer gets nulled out.

Move semantics are also supported by most if not all of the standard library types where appropriate, such that moving a vector costs no more than copying 3 pointers, and similar with moving a string.

Improvements in the standard that came in C++11, C++14 and C++17 (often to catch up with optimisations that all major compilers have done for decades) have also meant that move semantics are not even necessary in a lot of cases:

* C++11 guaranteed at least return-by-move where possible and enabled returning of move-only types by value
* C++17 guaranteed return value optimisation (*even if the copy/move constructor or destruct have observable side effects*), which enabled returning even non-moveable types by value, as well as guaranteed copy elision when initialising a variable with a prvalue expression of the same type

  Return value optimisation:

  ```cpp
  T f() {
    return T(); //T is constructed in the calling function's stack frame, no copy or move operation or destruction occurs here
  }

  f(); // only one call to default constructor of T
  ```

  Guaranteed copy elision during initialisation:

  ```cpp
    T x = T(T(f())); // only one call to default constructor of T, to initialize x
  ```

## Value types

Value types are either directly a local value, or an object that transparently represents a value and will do a deep copy by default.

A value type may be an object with automatic storage, or objects with external memory that's encapsulated entirely by the type like a `std::string` or `std::vector`; when you copy one, it copies the contents too.

Knowing if a type manages memory elsewhere is important for performance characteristics and so on, but if it is written correctly and fully encapsulated and supports value semantics, then it is not important for ownership: all types value types represent ownership of their value.

C++ is an unusual language in that all types are value types by default (and passed-by-value) as it is more explicit for the programmer to control performance characteristics, but it can lead to some major performance issues if the programmer doesn't realise this.

Value types can make runtime polymorphism difficult to achieve, but there are tools to help with this, such as `std::variant`.

## Reference types

(Unfortunately, I am using the word "reference" here in it's English sense, not in the C++ sense)

Reference types refer to an object that may be somewhere else rather than having it directly.

This may be via a pointer, a C++ reference, an iterator, a smart pointer object, an object that does not do deep copies by default, etc.

Reference types are required when you need to read or modify an object that exists somewhere else and is also often used when that isn't necessary, but a value is expensive or not possible to construct, copy or destroy.

Reference types may or may not model ownership, it is up to the programmer to choose a reference type which models the kind of ownership that is appropriate for the situation.

Most popular languages use reference types (and pass-by-reference) by default, though their compiler or runtime may optimise these to become effectively the same as if they were value types. It's very hard or impossible to force something to be a value in these languages and as such it can be very hard or impossible to give performance guarantees.

Reference types make reasonable performance easier to achieve naively, and also allows for easy polymorphism.

C++ allows reference types and pass-by-reference, but doing so is explicit.

## Optional values

Sometimes a value, such as a function parameter or return value, can exist or not exist. Because of RAII, we need to have types which have a state that can represent no value, for the cases where the value does not exist.

Traditionally, a raw pointer is used to represent optional values, it is set to `nullptr` when it doesn't exist, and a valid pointer to an object when it does exist.

A C++ reference cannot be used for optional semantics, because a C++ reference cannot be null (without invoking Undefined Behaviour).

Smart pointers, particularly the standard ones (`std::shared_ptr` and `std::unique_otr`), are able to represent optionality also; they can have a value of `nullptr` which represents that it is not pointing to an object. This is even their default constructed state.

C++17 introduced `std::optional`, and earlier some 3rd party libraries introduced equivalent types, which are basically a class which combines a flag and a value. The flag is set when the value is set, and the value can be retrieved from the optional object.

These types better represent what is happening to future readers of the code as you explicitly call out that the value is optional, whereas a raw pointer may be there for a number of different reasons. Raw pointers also necessitate an indirection, where `std::optional` and its equivalents are able to store the value internally.
`std::optional` is therefore a value type, where a raw pointer is a reference type.

The types often have a mechanism for making sure that the value exists before allowing access to it.

## Automatic Storage and the Free Store

Automatic storage is what the C++ standard calls "things that are on the stack", except that it also includes things like direct non-static data members, which might be in the heap if their containing class instance is on the heap, like a member of an object in a `std::vector`.

Things in automatic storage are necessarily uniquely owned by the enclosing scope, whether that be a function or a class. This makes it a very strongly preferred default place to store objects, on top of the performance advantages of cache coherency, lack of indirection, and transparency to optimisers - this last point means that the compiler is allowed to put something which is in automatic storage into a different scope from which it is initialised - for example, in Return Value Optimisation.

Free store is what the C++ standard calls the heap. It uses the term "free store" because the C++ standard does not specify that the machine must have a stack + heap memory model, so uses this agnostic term to mean "wherever the runtime determines is an appropriate place to allocate things".

Things in the free store need to have their ownership carefully managed by the programmer, preferably by using a type that models ownership.

### Static storage

Static storage is uniquely owned like automatic storage, but has a different lifetime, which depends on what type of static it is.

Static storage has a lot of the same issues as regular global variables (in fact, globals are often static), so it is still best to avoid this type of storage where possible.

## Ownership vs Aliasing

Ownership is *not* about how many people/things can point to an object. Having a `std::unique_ptr` pointing to something **does not mean** you're not allowed or shouldn't have other pointers or references to that same object. Having multiple ways of referring to a single object is called "aliasing".

Ownership does mean that you should not have any other *smart* pointers pointing to the object, however.
This is because smart pointers represent *ownership* of an object, and a `std::unique_ptr` is supposed to have exclusive ownership of the thing it points to.

Having a raw pointer or reference pointing to the thing owned by a `std::unique_ptr` does not change anything about ownership, and so is completely fine - as long as you don't do anything ownership-related with the raw pointer or reference.

## C++ types that model ownership

* automatic storage / local variables

  ```cpp
  void func(){
    int thing; //func's stack owns thing
    float* ptr; //func's stack owns ptr, but not the float that it points to
  }
  ```

* `std::unique_ptr`

  ```cpp
  void func(){
    std::unique_ptr<Object> obj = std::make_unique<Object>(); //func's stack owns obj and through that, the Object it points to
  }
  ```

  `std::unique_ptr` will automatically call `delete[]` for array types and `delete` otherwise.

* `std::shared_ptr`

  ```cpp
  void func1(std::shared_ptr<Object> o){
    while(true);
  }

  void func2(std::shared_ptr<Object> o){
    while(true);
  }

  void func3(){
    auto o = std::make_shared<Object>();
    std::thread t1(func1, o);
    std::thread t2(func2, o);
    //Here, func1, func2 and func3 all share ownership of the object pointed to by o

  } //irrelevant: this will crash here cos the threads weren't joined
  ```

  Unfortunately, `std::shared_ptr` is only finally adding support for C-Style array types (`T[]`) in C++20. It's best to use another alternative where possible anyway, like `std::array`, instead of c-style arrays.

* types with value semantics (`std` collections, `std::optional`, `std::variant`, `std::any`)
  * note that ownership modelling is still lost if the type contained in these is a type that does not model ownership, or more pedantically, the containing type only owns the contained object, not anything it might refer to, so it is up to the contained object to model ownership if it needs to

  ```cpp
  std::vector<int> ints; //ints owns the integers it contains
  std::vector<Widget*> widgets; //widgets owns the pointers, but it does not own the pointed-to objects
  std::vector<std::unique_ptr<Widget>> widgets2; //widgets2 owns the unique_ptrs, and through those, owns the pointed-to objects
  ```

* static variables - although this is almost too weak to include in the list

  ```cpp
  namespace {
    int blah; //This variable will be created when the module (exe/DLL) this TU is in is loaded and will be destroyed when the module is unloaded
  }

  void func(){
    static float blah2; //This variable will be created when func is first called and destroyed when the module this TU is in is unloaded
  }
  ```

* `std::auto_ptr` - deprecated in C++11, removed in C++17, because it didn't have move semantics, resulting in it being very hard to impossible to use correctly

  ```cpp
  std::auto_ptr<double> doNotUse(new double(3.0)); //Owns the double, but should not compile in c++17 onwards
  ```

The above are in *order of preference*

## C++ types that do not model ownership

* raw pointers

  ```cpp
  Object* bad(){
    Object* bad_owning_object_ptr = new Object(); //This object is owned by p_object but that ownership is not modelled by the type of p_object, Object*
    // someone needs to remember to clean up the object at some point
    return bad_owning_object-ptr;
  }
  ```

* references

  ```cpp
  void func(Object& o){ //able to refer to and even modify `o`, but does not model ownership of `o`
  }
  ```

* `std::weak_ptr` - though they allow you to gain ownership later, safely

  ```cpp
  void func(std::weak_ptr<Object> o_weak){

    auto o = o_weak.lock(); //o is a std::shared_ptr<Object>

    if (o)
    {
      o->doSomething;
    }
    else
    {
      std::cout << "already destroyed\n";
    }

  } //If it managed to lock the weak_ptr, the object pointed to by o will be cleaned up here

  void func2(){
    std::thread t;

    { // new scope for the sake of illustrating the point

      auto o = std::make_shared<Object>();
      t = std::thread{func, std::weak_ptr<Object>{o}};

    } //o is destroyed here, but the Object it points to might live on if the thread has locked its weak_ptr by now. If it hasn't been locked, it will be cleaned up here.

    t.join(); //This is basically a "safe" race condition between the thread locking the weak_ptr -> shared_ptr and the end of the scope above. It will either own or it won't, it definitely won't delete twice or forget to delete

  }
  ```

  This isn't exactly good code, it's just to demonstrate that `std::weak_ptr` does not keep the object it refers to around if it would otherwise be cleaned up, but that it can be used to safely detect if the object has been cleaned up, even from another thread

## Leaking memory with smart pointers in C++

Using smart pointers doesn't mean you no longer have to think about ownership at all, it just makes sure that you're thinking about certain aspects of ownership. It is still possible, for example, to leak memory with `std::shared_ptrs`:

```cpp
struct Blah{
    std::shared_ptr<Blah> other;
};

void leak(){
    auto a = std::make_shared<Blah>();
    auto b = std::make_shared<Blah>();
    a->other = b;
    b->other = a;
} //The two Blah objects will still exist, despite a and b being cleaned up, because they both keep a reference to the other, holding the refernce counts above zero
```

It's quite possible to do the same with `std::unique_ptr`, but it is a little bit harder to do without it being obvious that you're playing with fire:

```cpp
struct Blah{
    std::unique_ptr<Blah> other;
};

void leak(){
    auto a = std::make_unique<Blah>();
    auto b = std::make_unique<Blah>();
    a->other = std::move(b);
    a->other->other = std::move(a);
    //Really the only hint is the extra level of indirection, and the fact that if we did it like the shared_ptr example, without the extra indirection,  we'd get a crash every time for derefencing the now null `b`.
}
```

## How to fix

* Never allow cycles of ownership

  * luckily, it's extremely rare that a design with ownership cycles is the most accurate model of a given problem, so usually can be avoided by just re-examining your architecture, but you do have to realise you have a cycle

* Have a GC that supports reachability analysis (pretty much all GC'ed languages have this nowadays?)

  * this is where the runtime will traverse from any global variables and scopes, down through any pointers/references it has, and "mark" what it comes across. When it is finished traversing all that it can reach, it will clean up anything that it hasn't marked.

  * I don't know of any such types / mechanisms in C++

## Double delete with smart pointers in C++

It is also very possible to still have double delete issues even using smart pointers, but it's a little harder to do accidentally using the standard smart pointers:

* `std::shared_ptr`

  ```cpp
  {
    auto ptr1 = std::make_shared<Object>();
    auto ptr2 = std::shared_ptr<Object>(ptr1.get()); //Both shared_ptrs point to the ssame object, they both think they own it.
  } // Double delete of the object here
  ```

* `std::unique_ptr` is basically the same:

  ```cpp
  {
    auto ptr1 = std::make_unique<Object>();
    auto ptr2 = std::unique_ptr<Object>(ptr1.get()); //Both unique_ptrs point to the same object, they both think they own it.
  } // Double delete of the object here
  ```

  It is possible to tell a `std::unique_ptr` to give up ownership of an object by calling the `.release()` member function. This resets the pointer to `nullptr`, but does not destroy the object.

## How To fix

* When using `std::shared_ptr` to share ownership of an object, always use the copy constructor or the copy assignment operator of `std::shared_ptr`, never the constructor or reset function with a raw pointer to the object

  ```cpp
  void func(){
    auto objptr = std::make_shared<Object>();
    auto objptr2 = objptr; //Do this

    auto badptr = std::shared_ptr<Object>();
    auto badptr2 = std::shared_ptr<Object>(badptr.get()); //Not this
    badptr2.reset(badptr.get()); //And definitely not this
  }
  ```

* When transferring ownership from something to a smart pointer, make sure that the ownership is given up by the previous owner at the same time, especially if the old owner is also a smart pointer.

  ```cpp
  void func(){
    std::unique_ptr<Object> o_ptr{getObject()}; //c API that gets a pointer to an Object, ownership passed with raw pointer (because C has no better way). This is fine.
  }

  void preCpp14(){
    std::unique_ptr<Object> o_ptr{new Object()}; //Before C++14 brought make_unique, we had to use the constructor of unique_ptr and pass in the raw pointer. This was unsafe if and only if there is more than one expression that can throw (like a `new`) in a single statement.
    //The code above is fine pre C++14 as long as the caveats are minded, but post c++14, we should use make_unique, unless we need a custom deleter or are accepting ownership of an existing raw pointer.
  }
  ```

  * An example of problematic code from Herb Sutter's GotW:

    ```cpp
    void sink( unique_ptr<widget>, unique_ptr<gadget> );

    sink( unique_ptr<widget>{new widget{}},
          unique_ptr<gadget>{new gadget{}} );
    ```

    Here, there is no guaranteed order between the two `new`s and the construction of the `std::unique_ptr`s. The first new could succeed, and the second one throw, before either unique_ptr is constructed, which will leak the first `widget`.
    Switching to `std::make_unique` completely eliminates this issue.
  
  * sometimes it is not possible to use `std::make_unique`, such as when providing a custom deleter, or when taking ownership of a raw pointer from a C API. In such cases, the construction of the `std::unique_ptr` should be done inside a function. This guarantees that only one expression that can throw is in any one statement that has unspecified sequence order (like function parameters)

    ```cpp
    void sink( unique_ptr<widget, CustomDeleter<widget>>, unique_ptr<gadget, CustomDeleter<gadget>> );

    std::unique_ptr<widget, CustomDeleter<widget>> getWidget(){
      return std::unique_ptr<widget, CustomDeleter<widget>>(getOwningRawWidgetPointer(), CustomDeleter<widget>());
    }

    std::unique_ptr<gadget, CustomDeleter<gadget>> getGadget(){
      return std::unique_ptr<gadget, CustomDeleter<gadget>>(getOwningRawGadgetPointer(), CustomDeleter<gadget>());
    }

    void doThings(){
      sink(getWidget(), getGadget()); //These are now exception safe
    }
    ```

* Never make a smart pointer point to something that's also owned by something else other than through the above 2 methods, unless using a custom smart pointer that specifically supports it.

## What type to use

* in practice, almost never `shared_ptr`

### Function Parameters

* is the parameter optional?
  * two choices
    1. use `std::optional` and follow the rest of this guide to choose the parameter type for inside the optional (using `std::reference_wrapper` instead of `&`), or
    1. overload the function with a version without the parameter

* Does someone in the calling scope have ownership, or is the object owned by something that will definitely survive longer than this function scope?
  * Does someone in a child scope need to take ownership of the object - i.e. does this function or a child function need to spawn a thread or deferred task that uses something that isn't guaranteed to survive long enough?
    * Does the current, parent or concurrent scopes also need ownership of the object?
      * Take by `const std::shared_ptr&` and then take a copy at the scope where it is needed
        * Note: that a const reference to a `std::shared_ptr` does not in itself have any ownership - it must be copied before ownership is taken. It is simply used as a way for child functions to be able to get a copy if they need one
    * else
      * take by `std::unique_ptr` and force calling side to use r-value or `std::move` arguments in
  * else
    * Is the object larger than a pointer or two, or otherwise non-trivial to copy?
      * Do you need to take a copy of the parameter at this scope?
        * parameter is passed by value
      * else
        * do you need to be able to modify the parameter in-place / the same instance?
          * take by `&`
        * else
          * take by `const&`
    * else
      * do you need to be able to modify the parameter in-place / the same instance?
        * take by `&`
      * else
        * pass by value
* else
  * does the current owner need to keep ownership of the parameter?
    * take by `const std::shared_ptr` (non reference)
  * else
    * take by `std::unique_ptr` and force caller to `std::move` into the argument into the function

#### Notes

* "What about raw pointers?"

   It's ok to take a raw pointer as a parameter, if you need reference semantics and as long as it is non-owning (i.e. a replacement of any of the reference-based options), but also consider using a `std::optional` (C++17) instead if that's why.

  It's always better to encode your constraints in the type, and for this reason, a raw pointer is almost never the right choice for a function parameter, as it has basically zero constraints

  If the parameter is owning or does not require reference semantics, I strongly recommend using another type.

* "What about output parameters?"

  They are basically never a great idea
  
  * "But I have multiple return values"
  
    return a struct or tuple instead.

    std::make_tuple and C++17's structured bindings help with brevity.

  * "But I need to ensure performance by making the object in the parent scope and using an output parameter to fill it in"

    This can make it harder for the compiler to optimise away the object when nesting deeper

    It also makes it impossible to have the parent scope local variable be const.

    Return-by-at-worst-move has been guaranteed by C++11, RVO has been guaranteed by C++17, and major compilers have done both since before C++98.

  * "But typing `std::optional<std::reference_wrapper<BigLongTypeName>>` or similar everywhere is a pain in the butt"

    Use your IDE and/or return type deduction.

### Function return values

* return by value by default
* rely on RVO to elide copy (don't return by `std::move` unless necessary, as it disables RVO)
* if reference semantics are necessary
  * if you're sure that the caller will not use the return value outside of the lifetime of the thing returned, and the thing returned will not move in memory
    * return by (preferably `const`) `&`
  * else
    * return by `std::shared_ptr`
* if you know the object is going to be stored in the heap (e.g. for runtime polymorphism)
  * return by `std::unique_ptr` / `std::make_unique`
* if the return value is optional, use a `std::optional`. This is a stronger recommendation to avoid a raw pointer to represent optional semantics than for a function parameter, because there's potentially an unknown number of places where the value might be used without checking - which `std::optional` makes (slightly) more difficult to do accidentally.

#### A note on return type deduction

`auto` will never deduce a reference (though it can deduce a pointer), and return type deduction is no exception. If you need to be able to deduce a reference type, use `decltype(auto)` instead. This is rarely what you want, though, and is here as a note to those interested in writing very generic code (templates / TMP)

### Local variables

* naked type is best
  * order of preference for const-qualifiers is `constexpr` -> `const` -> non-`const`
  * implied unique ownership thanks to RAII
  * simplest for compiler to optimise
  * "can't" be modified outside of this function's scope
  * lifetime solely managed by this function's scope

* `auto&&` can be used (thanks to lifetime extension) for zero cost splitting of statements, useful for debugging, though I do believe it would still allow intermediate temporaries to be destroyed, if that matters in your use case
  * release mode is still going to compile the variable out

* If the naked type is too large to put on the stack, or you know it will need to be passed around and is expensive to pass by value, then use a `std::unique_ptr` if possible. Switch (via `std::move`) to a `std::shared_ptr` only at the point where shared ownership is really required.
  * It's possible to go from `std::unique_ptr` to `std::shared_ptr`, but not the other way around (not any more, anyway, because it was realised that it was basically unsafe to do so)
  * Again, remember that RVO, including NRVO, is a thing, so even if you return it by value, it should not do any unnecessary copies.

* if you need reference semantics
  * preference order is
    * `&` if you know you don't need ownership
      * `*` if you know you need to reseat the pointer (point to something else)
        * often moving the guts of a loop to a function / lambda can let you use a reference instead, if the body of the loop is big enough for that to be a good idea anyway
    * `std::unique_ptr` if you need ownership and unique ownership is possible
    * `std::shared_ptr` if you need to share ownership
  * wrap in `std::optional` if it is optional
  * use order of preference for constness is `constexpr` -> `const` -> non-`const`

### Class non-static member variables

* this is basically the same as the local variable section, except that instead of thinking about lifetime of the scope of the function, you have to think about the lifetime of the object you're in.
* it's still far more common than most people seem to realise that it is OK to use references (although this disables move and copy operations) instead of shared ownership.
  * If you need move and copy operations, try `std::reference_wrapper` or just use a raw pointer - just make sure to initialise it to `nullptr`!
* Also not a good idea to rely on lifetime extension here, as that only works within scopes, not object lifetimes

### Lambda captures

This is exactly the same as with a non-static data member, as that's all lambdas are: syntactic sugar for defining and instantiating a class inline.

Just make sure you notice that when you're passing a lambda to a callback or thread, the captures need to be available at the point of calling the callback or thread, so make sure you think about lifetimes and ownership and guarantee it with `std::unique_ptr` or `std::shared_ptr`.

Note also that copyability and moveability of captures affects the copyability and moveability of the lambda (as it would any class), so be sure to think about where you're passing it to and how.

Sometimes it is necessary to mark a lambda as `mutable` e.g. `[]() mutable {}`, as lambdas are immutable by default (unlike a class).

### Static

Static lifetimes last until module unload, so do carefully consider if this has any implications on your objects and references.

Relying on the automatic cleanup of statics means the cleanup code will run at a non-deterministic time after `main()` has completed (or DLL has unloaded), so it can make debugging difficult.

### Interfacing with C APIs

* Make sure that it is clear when ownership is transferred across the API boundary.
  * Be aware that memory allocated by one module is often not safe to release from another module, particularly if the compiler, linker, runtime, SDK, etc, settings are different between them.

* Strongly **avoid sharing ownership** across the API boundary - if you must, make sure there's a way to detect/signal that ownership is no longer needed

* Use smart pointers within C++ code, and `release()` to get the pointer to pass to the C-API at the last possible moment as ownership is transferred.

### Older C++

For the case of non-owning optional semantics before C++17, use something like `boost::optional` if you are able, or fall back to using a raw pointer if you must.
**Make sure that you are careful with it.**

For pre-C++11, using a 3rd party library, or, if you must, write your own, just be aware that move semantics are not available so move-only types are not possible (this is why `std::auto_ptr` is deprecated/removed).
Falling back to raw pointers is not recommended at all. Even a `typedef` to put "owning" in the type name would be better than nothing.

### Summary of type choice

* Do you *need* ownership?
* Do you *need to share* ownership?
* Do you need to be able to **modify** the object?
* Do you need to be able to **reseat** a reference-like thing?
* Do you need to be able to **give** someone else **ownership**?
* Is the object **expensive to copy** or too large for the stack?
* Is the object **optional**?
* Do you need to be able to **detect if an object still exists, without holding ownership**?
* What **version of C++** are you targeting? Can you use boost?

## Gotchas

* Ownership cycles as mentioned already

* Naïve implementation of list/tree based data structures using smart pointers to represent ownership can lock up or even crash (stack overflow) during clean up - the destructor of each node calls the destructor of the things it owns
  * Set a custom deleter of the smart pointer or destructor of the node type that traverses rather than recurses, either freeing their resources as you go, or disconnecting the nodes by moving things to a 'to delete' collection and freeing their resources later when you have spare time or in another thread.

* Interactions between smart pointers and forward references
  * Complete type is required for `std::unique_ptr` at the point of the destructor being called
    * This is in the header of a class definition if the destructor is not user defined and the class has `std::unique_ptr` members
    * Say you have a class A and a class B. Class B has a `std::unique_ptr<A>` data member, but consumers of class B should not need the full definition of A. B has no need to have a user defined destructor.
    Declare a ~B() destructor in B's header, and define it as explicitly defaulted in the .cpp file.
    If B does need a user defined destructor, similarly, you need to either have the full definition of A before the destructor is defined, or have ~B() defined in the .cpp file (not inlined).
    * Unfortunately, this means you can't inline the destructor of B, unless you fully define A.

  * Complete type is required for `std::shared_ptr` at the point of construction or setting of the `std::shared_ptr`
    * This has similar implications to `std::unique_ptr` but applies to construction instead; any constructor that is going to initialise the `std::shared_ptr` member to an actual instance needs to either not be inlined, or needs to have the full definition available in the header.
    * Defaulted constructors have less of an issue because a default constructed `std::shared_ptr` does not construct an instance of the held type, whereas a defaulted destructor of a `std::unique_ptr` does need to know the definition of the destructor of the held type

  * Basically, if you have smart pointer data members, don't inline your user defined constructors and don't inline your destructors, or have slow builds by including the definition of the pointed-to types

* Default arguments for function parameters are constructed at the point of calling them
  * Just overload the function instead (inline if you have to), default arguments have too many gotchas
    * e.g. overriding a function with a default arg does not let you define a new default argument, unless
      * you call it by pointer to derived, or
      * you cast the function to another declaration with a new default argument.

* Pay attention when making non-owning objects - especially in expressions involving temporaries - to make sure they're not going to dangle beyond the life of the object they refer to:

  ```cpp
  void bad(){
    const char* c_str = (String("Hello") + " World!").c_str();
    //c_str now points to memory that this program does not own, as the strings are cleaned up and the `const char*` does not model ownership
  }
  ```

  ```cpp
  void bad(){
    std::string_view sv = std::string("Hello World!");
    //sv is dangling for the same reason, string_view is a non-owning reference type
  }
  ```

* l-value versus r-value is not sufficient to determine temporariness:

  (thanks to [Arthur O'Dwyer](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/))

  It has been claimed that you should write classes and interfaces that distinguish between temporary and non-temporary objects by using r-value and l-value qualified functions and parameters.

  For example, `std::string` has a deleted `operator std::string_view &&`, to try to stop you making a `std::string_view` that points to an r-value `std::string`. However, this disables some perfectly fine, valid use cases:

  ```cpp
  void print_it(std::string_view param) {
    std::cout << param << std::endl;
  }

  void test() {
      use_it(String("hello world"));
  }
  ```

  On the other hand, it is also possible to make an example of code that deals with temporaries that are never r-values:

  ```cpp
  TokenIterator make_iterator(const std::string& haystack) { //There is a temporary `std::string` that is not an r-value
    return TokenIterator(haystack);
  }

  void test() {
    auto tk = make_iterator("hello world");
    auto first_token = *tk;  // compiles and CRASH!
  }
  ```

  This basically means that you can't entirely get away with having types and interfaces that make it completely unnecessary to think about ownership, in C++.

## Common dissenting remarks

* "I don't like ownership and move semantics so I'm going to stick to raw pointers, `new` and `delete`"
  * AKA "I've been doing this for blah decades and never had a problem!"
  
  This doesn't make the concept or the problems go away, it just makes dealing with it manual - instead of allowing the compiler to help and check.

  Have you looked at how many CVEs are the result of people using memory incorrectly in C and C++?

* "Smart pointers are slow"

  * `unique_ptr` is not slow
    * Not reference counted at all
    * (almost) all of its useful code runs only at compile-time
    * there is only one case I know of where it isn't a truly zero-overhead-abstraction (at runtime), which is deleting a moved-from `std::unique_ptr` of a non-trivially-destructible type. I'm not 100% sure it's even possible to write generic code that doesn't have an equivalent cost while maintaining correctness

  * `shared_ptr` is rarely too slow if used correctly - that is, only when shared ownership is truly necessary. This is very rare in my experience
    * using `make_shared` allocates memory for the object and the reference counting block at the same time - ensuring cache locality and minimising OS calls to allocate
    * the reference counting block itself is thread-safe but the access to the object and the `shared_ptr` itself are not
      * This means it is threadsafe to make a copy of a `std::shared_ptr`, but not thread safe to share a `shared_ptr` by reference between threads, or to rely on the `std::shared_ptr` to serialise access to the underlying object
    * AFAIK, the only way to do this faster yourself is to give up shared ownership, give up correctness, or give up ref-count thread safety (this last one *might* actually have a valid use case).

## Take-aways

* Think about who needs to delete something and when

* Almost never use `new` and `delete` keywords directly - unless you're writing a smart pointer, and even then, almost never

* Use the most constrained type you can in a given circumstance

* Don't take a parameter by smart pointer or even reference to smart pointer unless you specifically need to be able to manipulate the ownership of the object. Just use a (`const`) reference instead.

* Nothing *wrong* with raw pointers - as long as they're **non-owning**