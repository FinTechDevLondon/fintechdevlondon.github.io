---
layout: post
title: Type Erasure with C++, A Comprehensive Review
---

## Type Erasure with C++, A Comprehensive Review

C++ is a statically typed language. At the most basic level, this means that something of one type cannot be put into a place where something of another type is expected. For example, the following doesn't appear to make any sense in the C++ language.

```cpp
class MyType;
class AnotherType;

int main()
{
    MyType myType = // something;
    AnotherType anotherType = myType; // error, does not make sense, the type is wrong
}
```

By contract, in dynamically typed languages such as Python it is possible to assign something of one type to a variable which was previously referencing something of another type.

```python
class MyType:
    
    def MyType(self):
        self.y = 1
    
class AnotherType:
    
    def AnotherType(self):
        self.x = 2.0

variable = MyType()
variable = AnotherType()
variable.x = 10.0

print(variable.x) // prints '10.0'
```

Python is dynamically typed to such an extent that the variables `x` and `y` internal to the two classes don't really exist unless we create them dynamically. There is no way in Python to say *I have a class and it has 2 integers and a floating point as member variables*. Instead, we simply use the object as if it has whatever variables we want inside it, and those will be created dynamically.

This means that C++ and Python are good for two very different ideoms of programming. On the one hand, Python is flexible. If we want to store multiple objects of different types in a list, then this can be done trivially. On the other hand, Pythons implementation of this is inherently error prone. What happens if we have a list containing several objects, and we try to call a particular function on each of the objects in the list?

For example, consider the following. Imagine we are writing a game, and we have a set of entities that represent drawable objects. These might be the players character, some enemies and some components of the environment, such as trees and buildings. We can put these all in a list, and provided each object has a `draw()` function, we can iterate over the list of objects invoking `draw()` on each of them.

But what happens if we need to invoke some other logic? The player might have some functions for movement - maybe a `jump()` function. Trees and buildings don't jump, so presumably they don't define such a function. What happens if we need to call such a function? If we try to invoke `jump()` for every element of the container, we will encounter an error, and we won't know about it until runtime. In C++, the static type system would have caught this error at compile time.

We can get around the Python problem by doing some kind of check.

```python
// Pseudocode
for each element in the_list:
    if element.type() == something_that_can_jump:
        element.jump()
```

But Python is already slow, which is why it isn't used to write the logic for AAA games, and we just made it slower by adding an additional level of branching.

It would be nice if we could harness the speed of C++, but also be able to use Python-stype duck-typing in our code. With use of the techniques of Type-Erasure, we can have both.

### What is Type-Erasure?

There appears to be a significant amount of confusion as to what Type-Erasure in C++ *exactly is*, *how it should be implemented* and *what problems it solves*. There are numerous blog posts online which present different things, most of which I would not describe as *real* type-erasure. There are likely several reasons for the confusion.

- Different authors wanted to achieve different things, or solve slightly different problems. This can lead to a situation where one proposed example of type-erasure isn't suitable for a different application. Because the application and problem intended to be solved usually isn't explained, this can lead to a situation where it isn't immediatly clear what an example code can do and what its limitations are. It could be argued that there are different "levels" of type-erasure and that one solution incorporates more features than another.

In this review, I define type erasure in the following way.

> I have a vector, or other container, and I want to be able to store both `int`s and `double`s inside it, and then by extension, any other type `T`

This obviously isn't something afforded by `std::vector<T>`, which contains a specific and fixed type `T`.

### A Non-Solution

When the problem of type-erasure arrises in real-world situations, many software engineers default to a solution which I describe as a non-solution. It works, in the sense that the code compiles and runs, but the code which is written doesn't describe the problem at hand or the solution to it.

Jumping into it - how many times have you seen something that looks like the following?

```cpp
std::vector<OneType> vecOfOneType;
std::vector<AnotherType> vecOfAnotherType;
std::vector<YetAnotherType> vecOfYetAnotherType;

enum class MyTypes
{
    OneType,
    AnotherType,
    YetAnotherType
};

if(someObject.type() == MyTypes::OneType)
{
    vecOfOneType.push_back(someObject);
}
else if(someObject.type() == MyTypes::AnotherType)
// ...
```

There are a multitude of problems here.

- Repetition. We had to introduce 3 vectors when there should only really be one container for our objects.
- Extra logic. Everywhere we want to use one of these containers, we have to repeatedly write out a long `if-elseif-else` block covering all the possible types. These additional branches will inevitiably slow the code down, so if possible we seek a solution which doesn't have this.
- Maintainance. Every time we want to add or remove a type, we incur a significant maintainance bill. The `if-elseif-else` block might be used in hundreds of locations throughout our code. It will be costly to update all of them. It will also be risky, because we might miss some occurances, which will likely lead to silent failures or undefined behaviour.
- It isn't elegant. Good code should be elegant. It should look "nice" and be enjoyable to work with. This "feels" like a "hack" or a poorly designed workaround to a language restriction - because it is. Do you want to work on something like this? Probably no.

There is another problem, but this one is slightly more subtle.

- Lack of a deeper inferred meaning or statement of intent.

The code written doesn't say anything about the problem beyond that of the surface level details. There is no "meta-conversation" which occurs at a higher level above what is written. The code here says - literally - that we have a container of one thing, and a container of another thing, and a container of yet another thing. Although these lines happen to occur one after the other, the code suggests that they are seperate things in independent containers. In reality this is wrong - the opposite is true. We *intend* to use these containers together - as if they were one container. So why haven't we written code which *says this*.

> Code should infer some broader or higher level meaning to the reader, not just be a surface level description of a recipie which solves the particular problem at hand. This is what differentiates software engineers who are good at writing libraries from those who can just turn the crank and produce a solution to todays problem, something which isnt' adaptable to tomrrows.

### Dave Kilian Type Erasure

Although the problem intended to be solved has already been stated, namely being able to store different types in the same container, first different forms of type-erasure will be explored.

Dave Kilian has written a blog post on the subject of type erasure. It can be found [here](http://davekilian.com/cpp-type-erasure.html). Unfortunatly I wasn't able to find the original source on Hacker News.

Dave covers two cases, firstly using polymorphic interfaces, and secondly using templates. The first case looks something like this

```cpp
class Animal
{
    public:

    virtual std::string see() const = 0;
    virtual std::string say() const = 0;
};

class Cow : public Animal
{
    public:

    std::string see() const override
    {
        return "Cow";
    }

    std::string say() const override
    {
        return "moo";
    }
};

class Dog : public Animal
{
    // ...
};

std::vector<Animal*> animal_vec;

void someFunction(const std::vector<Animal*> animal_vec)
{
    for(const auto animal_p : animal_vec)
    {
        std::cout << "Animal " << animal_p->see() << " says " << animal_p->say() << std::endl;
    }
}
```

There are a multitude of problems here.

- Reference symantics. We are forced to store a pointer when we would rather have stored a value.

Our container requires that we store the same type of object, and in particular these objects must have the same size in memory, or the container cannot allocate memory for them, since the size of each element must be known in advance, and must be the same. Although it looks like we are storing something which can be a different type, actually we store only a pointer. It is always 8 bytes wide, and points to some heap allocated memory which contains the runtime information required for virtual function dispatch.

- Tightly coupled inheritance hierachy. We are forced to have a system where everything which may be stored in the container must inherit from the interface defined by `class Animal`.

Although our inheritance hierachy might be shallow and wide, we will inevitiably have strong couplings in places which make little sense. We might have another inheritence hirachy which already exists. If we want to store these objects in the container, they must additionally inherit from an interface class. Anyone with even a moderate amount of experience with C++ will already anticipate the maintainance nightmare that will unfold if we continue this path.

Also consider the "is-a" vs "has-a" ideom. If you have a collection of unrelated objects which you wish to store in a container, then what is the "is-a" relationship? In the best case, they are all instances of some theoretical abstract concept which doesn't exist and has no tangable real-world concept. In which case, what has this inheritance relationship gained us, other than additional confusion and complexity? The code is harder to understand.

The second example in Dave Kilian's write up uses templates to instantiate multiple versions of a function for different types. This isn't related to type-erasure, because it isn't related to storing multiple objects in a container. Therefore I do not discuss this in detail.

The next example from Kilian uses wrapper classes to forward function calls to what are now assumed to be unrelated types. This is a step in the right direction conceptually, because now we are assuming `class Cow`, `class Dog`, `class XYZ`, `...` are all unrelated types. They do not inherit from a common base class or interface.

Kilian then writes a second inheritance hierachy which wraps the unrelated objects.

```cpp
class DogWrapper : public AnimalWrapper
{
    Dog my_dog;

    public:

    const std::string see() const
    {
        return my_dog.see();
    }
}
```

We have gained the concept of wrapping types which are totally unrelated in a new inheritance hierachy which relates all the wrapper classes via an interface base class `AnimalWrapper`. However, this isn't a particularly useful step forward, because we now have the same problem as before. Namely, we are working with reference symantics, and we have to maintain a very wide hierachy, with one wrapper class for each type. Note that the pattern defined by the wrapper hierachy is just an instance of the Adapter Pattern.

Kilian presents the following code in the final section.

```cpp
class SeeAndSay
{
    // The interface
    class MyAnimal
    {
    public:
        virtual const char *see() const = 0;
        virtual const char *say() const = 0;
    };

    // The derived type(s)
    template <typename T>
    class AnimalWrapper : public MyAnimal
    {
        const T *m_animal;

    public:
        AnimalWrapper(const T *animal)
            : m_animal(animal)
        { }

        const char *see() const { return m_animal->see(); }
        const char *say() const { return m_animal->say(); }
    };

    // Registered animals
    std::vector<MyAnimal*> m_animals;
    
public:
    template <typename T>
    void addAnimal(T *animal)
    {
        m_animals.push_back(new AnimalWrapper(animal));
    }

    void pullTheString()
    {
        size_t index = rand() % m_animals.size();

        MyAnimal *animal = m_animals[index];
        printf("The %s says '%s!'", 
            animal->see(), 
            animal->say());
    }
};
```

Kilian seems to have somewhat missed the point here, although this does look similar to the correct code for type-erasure that will be presented later. Let's make an assessment of what is presented here.

- The private template class `AnimalWrapper` which inherits from the interface class `MyAnimal` does provide the basic structure needed for type erasure, although the template class stores a pointer to `const T` rather than storing `T` by value.
- The `SeeAndSay` class has additional functions which obfuscate the fact that it is the type erasure class we are aiming to write. It is possible to store `SeeAndSay` in a container, and each instance of `SeeAndSay` can store one or more instances of `MyAnimal` in the member variable `m_animals`.
- The fact that `m_animals` is a vector obfuscates things further. We would usually expect to store exactly one object in `SeeAndSay`, not a variable number of 0 or more.

We can improve things significantly by cutting the example down to make it clear.

```cpp
class TypeErased
{
    class MyAnimal
    {
    public:
        virtual const char *see() const = 0;
        virtual const char *say() const = 0;
    };

    template <typename T>
    class AnimalWrapper : public MyAnimal
    {
        const T *m_animal;

    public:
        AnimalWrapper(const T *animal)
            : m_animal(animal)
        { }

        const char *see() const { return m_animal->see(); }
        const char *say() const { return m_animal->say(); }
    };

    std::unique_ptr<MyAnimal> m_animal;
    
public:
    template <typename T>
    void setAnimal(T *animal)
    {
        // pseudocode ...
        m_animal = std::make_unique(animal);
    }
};
```

We can now store `TypeErased` in a container. Anything can be wrapped by `TypeErased` provided it supports the functions `see()` and `say()`.

But isn't this unneccessarily restrictive?

Yes it is. The requirement to support `see()` and `say()` further obfuscates matters. There will be some common operations that a type should support if it is to be placed into a container, but the functions `see()` and `say()` are not among them.

What operations should be supported? `TypeErased` should be

- creatable
- destroyable
- copyable
- moveable

As a consequence, `MyAnimal` has to support the same operations. We can clean things up further by storing `T` by value in `class AnimalWrapper`.

- TODO: why don't we need virtual function dispatch for any of the other operations beyond `copy()`?

```cpp
class TypeErased
{
    public:

    template<typename T>
    TypeErased(const T value)
        : m_animal(std::make_unique(AnimalWrapper<T>(value)));
    {

    }

    ~TypeErased()
    {
        // m_animal will be destoyed here automatically since it is a unique_ptr
    }

    TypeErased(const TypeErased& other)
        : m_animal(other.m_animal->copy())
    {

    }

    TypeErased(TypeErased&& other) // = default
        : m_animal(std::move(other.m_animal))
    {

    }

    TypeErased operator=(const TypeErased& other)
    {
        TypeErased(other);
        *this = std::move(other);
        return *this;
    }

    TypeErased operator=(TypeErased&& other) // = default
    {
        m_animal = std::move(other.m_animal);
        return *this;
    }

    private:

    class MyAnimal
    {
    public:
        virtual std::unique_ptr<MyAnimal> copy() const = 0;
    };

    template <typename T>
    class AnimalWrapper : public MyAnimal
    {
        T m_data;

    public:
        AnimalWrapper(const T animal)
            : m_data(std::move(animal))
        { }

        virtual std::unique_ptr<MyAnimal> copy() const override
        {
            return std::make_unique(AnimalWrapper<T>(m_data));
        }
    };

    std::unique_ptr<MyAnimal> m_animal;
    
public:
    template <typename T>
    void setAnimal(T *animal)
    {
        // pseudocode ...
        m_animal = std::make_unique(animal);
    }
};
```

The functions `see()` and `say()` have been exchanged for a function `copy()` which is a virtual constructor. Often this function would be called `clone()`.

As Kilian notes, the names of the functions for `TypeErased::` do not have to correspond to those for `MyAnimal::`.


## Why not Boost Any?

Can't use boost any, because object has to be cast back to something. We don't want to have an explicit cast in order to be able to use an object.

## Why no std::any?

Not sure yet. Check. TODO


## Cheinan Marks *Practical Type Erasure*

Talks about boost any and having to use a big if block with casts to actually use it with a hetrogenius container.

I don't think this is exactly true. It is true that the interface (operations which can be called on the object) must be defined in advance.

https://www.youtube.com/watch?v=5PZVuUzP34g

Requires Loki typeinfo.

The backend uses templated classes, and boost any is used as a method to transport from the various templated backend classes to the frontend which the user sees. The frontend is templated as well, so I don't fully understand the reason for using boost any.


## Klaus Iglberger *Breaking Dependencies: Type Erasure - A Design Analysis*

