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



// how to add an image
//![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.