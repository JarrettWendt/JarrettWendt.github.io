---
layout: post
author: Jarrett Wendt
title: Reinventing the Wheel
---

The C++ Standard Template Library is legendary. It's like all of those generic containers and algorithms you learned about in undergrad, but fully realized and optimized to the nines. Generic enough to work with almost any data, and useful for just about any application.

So why doesn't anyone use it?

I may not have any industry experience myself yet, but from talking with industry professionals, it seems that basically nobody uses stl. Just look at Unreal Engine. They've got `TArray` instead of `std::vector`, `TMap` instead of `std::map`, `FString` instead of `std::string`... all of these accomplish the same thing, and in many respects (such as how you cannot have a `TArray` of `TArrays`) they're not even as capable as their stl equivalents! So why did Epic reinvent the wheel?

I've just finished reimplementing three containers from the stl:
- `std::vector` - my version is called `Array` to differentiate between vector the container and vector the mathematical concept.
- `std::forward_list` - my version is called `SList` because `forward_list` is just too wordy.
- `std::unordered_map` - my version is called `HashMap` because, again, less wordy, and I believe this name provides more information into the algorithms behind the container than simply that it is unordered.

Now complete<sup>*</sup> with these implementations, I believe I have some credence to explain why their existence is warranted. I believe the answer to the title of this article comes in three forms:

## 1. Use Case
The Standard Template Library is perhaps _too generic_ for some. C++ is a language that encourages low-level optimization. So when tasked with creating a generic containers library, a balance must be made between functionality and performance. How often will a user need to access the back of a linked list? Storing a back pointer will cost an extra pointer of stack space as well as a small bit of extra computation to maintain it. Is the benefit of the added functionality of maintaining a back pointer so beneficial to those users who need it that it's worth the performance hit to those users who don't?

These are questions that are impossible to answer and those who implement the C++ standard library have done their best guess at assuming the use cases.

However, when the users are those in a certain specialized industry, and heck they may even be your coworkers sitting in the desk next to you, we can start to make some more educated guesses on how these containers might be used.

Following is a table of the byte-sizes of my different containers compared to the stl. This was compiled with MSVC in release mode. Your stl sizes may differ depending on your compiler.

|                      | container | iterator |
|---------------------:|:---------:|:--------:|
| `std::forward_list`  | 8         | 8        |
| `SList`              | 24        | 8        |
| `std::vector`        | 24        | 8        |
| `Array`              | 24        | 16       |
| `std::unordered_map` | 64        | 8        |
| `HashMap`            | 32        | 32       |

So it looks like in most cases, my containers/iterators are 8-24 bytes bigger than the stl's. That doesn't sound like much, but we're talking about the fundamental containers that applications will be built upon. These bytes will add up and could contribute to some serious bloat. Let's see what justifications I have then...

Starting with the singly linked list, the standard decided that they'd only maintain a head pointer. I decided a tail pointer would be nice as well as a `size_t` for a count of the number of elements. In this instance, there's not really any choice but to reimplement the container. If you were to be lazy and try to reuse existing code by composing `SList` of a `std::forward_list` you'd have no way to retrieve, store, and the tail pointer since the nodes are not exposed.

The lists' iterators are both the same size, a pointer to a node. An alternative implementation would be to also carry a pointer to the owner list so that you may verify you're performing operations on the correct container when passing the `iterator` into methods on `SList`. I in fact have this enabled in debug, but for release builds I keep the smaller data type.

Now for the contiguous containers. My `Array` and the stl's `std::vector` are the exact same size, maintaining a pointer address, an element count, and a capacity. I believe most stl implementations actually store three pointers: the starting address, the ending address, and the address of the last element. Either way, it adds up to the same size and provides the same functionality.

The iterator here is the first real contention point in my opinion. My iterator is twice the size of stl's because they only maintain a pointer to the 

## 2. Ease of Use
The last point was somewhat nitpicky. 8 bytes here and there realistically won't be much of an impact. Sure they can add up, but the thing slowing down your application or bloating your memory is likely to be found elsewhere. A real gripe against the stl, and I believe anyone who's used it in any meaningful capacity can attest to, is how convoluted it can be to use.

`std::vector` provides `push_back` and `pop_back` methods. Great. It's very clear what they do. They append and remove from the end of the container. Cool. Now what if I want to append or remove from the front? Just call `push_front` or `pop_front`? Nope! Those methods don't exist! If you want to insert at the front of the container you must do this:
```c++
v.insert(v.begin(), t);
```
And if you want to remove from the front of the container you must:
```c++
v.erase(v.begin());
```
Well that's annoying. To be fair, it's clear why there is no `push_front` or `pop_front` provided for `std::vector`. It's because it would be horribly inefficient. To insert something at the front, all existing elements must be shuffled back one space. Likewise to remove they must all be shuffled forward. By not providing a `push_front` or `pop_front` the C++ standard is actively attempting to curb poor programming practices and instead encourage the use of containers such as `std::deque` which is likely more suited to situations like these.

That's great and all for the n00bs who don't know what they're doing, but what about those of us who _insist_ we know what we're doing? Many times I've decided a `std::vector` would be the right container for an application because of that sweet sweet O(1) random access, but we _sometimes_ need to insert/remove from the front? I wanna be able to call `push_front` and `pop_front` gosh darnit!

The sane solution to this of course is to throw away all the hard work that went into designing and optimizing `std::vector` and roll out your own container that you wrote in a 48-hour non-stop energy drink fueled sprint over the weekend.

Of course not. The sane and easy solution is to just inherit from from `std::vector`, and your own `push_front` and `pop_front` which just calls `insert` and `erase`, respectively.

But this was just one example. What if you want to remove all elements matching a certain predicate? C# conveniently provides `List<T>.RemoveAll(Predicate<T>)`, so C++ probably has something similar, right? No? Enter the erase-remove idiom:
```c++
const auto it = std::remove(v.begin(), v.end(), t);
v.erase(it, v.end());
```
Now _that_ is ugly. And what the heck? You're calling some generic free function `std::remove` to do it? Which returns an iterator?

This highlights my personal biggest gripe with the stl. C++ is a language of patterns. It is in fact more focused on these patterns than Object Orientation, which can seem completely bizarre to programmers coming from C# or Java. What is undoubtedly the king of these patterns is the iterator pattern. Iterators are extremely powerful, and it's thanks to them that generic methods such as `std::remove` can work on almost any object. So from the perspective of code reuse, free functions like `std::remove` are beautiful. There doesn't have to be a separate `remove` method on every container now. That rocks, but it doesn't make it any easier to use.

I believe that libraries should convenience themselves to the users as much as possible. No novice using C++ is going to discover the erase-remove idiom on their own. They're going to type `myVector.` and scroll through the intellisense popup looking for the `remove` method that isn't there. Then they'll see `erase`, figure it must mean the same thing, and then go to StackOverflow asking why their code doesn't work.

Unreal's `TArray` provides a `Remove` method, and so does my `Array`.

So does all of this really warrant "Reinventing the Wheel"? Probably not. Again, the easiest/smartest/safest move is to just make your own vector composed of a `std::vector` with no other data members and provide wrappers to all the existing methods in addition to whatever convenience methods you desire.

## 3. Not-Invented-Here Syndrome

