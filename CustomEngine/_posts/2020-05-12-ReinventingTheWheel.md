---
layout: customEngine
author: Jarrett Wendt
title: Reinventing the Wheel
excerpt: The C++ Standard Template Library is legendary. So why doesn't anyone use it?
thumb: assets/images/strandbeest.gif
category: customEngine
---

<figure>
<img src="{{ page.thumb }}" alt="strandbeest" width="300">
<figcaption>strandbeest</figcaption>
</figure>

&nbsp;

The C++ Standard Template Library is legendary. It's like all of those generic containers and algorithms you learned about in undergrad, but fully realized and optimized to the nines. Generic enough to work with almost any data, and useful for just about any application.

So why doesn't anyone use it?

&nbsp;

&nbsp;

I may not have any industry experience myself yet, but from talking with industry professionals, it seems that basically nobody uses stl. Just look at Unreal Engine. They've got `TArray` instead of `std::vector`, `TMap` instead of `std::map`, `FString` instead of `std::string`... all of these accomplish the same thing, and in many respects (such as how you cannot have a `TArray` of `TArrays`) they're not even as capable as their stl equivalents! So why did Epic reinvent the wheel?

I've just finished reimplementing three containers from the stl:
- `std::vector` - my version is called `Array` to differentiate between vector the container and vector the mathematical concept.
- `std::forward_list` - my version is called `SList` because `forward_list` is just too wordy.
- `std::unordered_map` - my version is called `HashMap` because, again, less wordy, and I believe this name provides more information into the algorithms behind the container than simply that it is unordered.

Now complete<sup>*</sup> with these implementations, I believe I have some credence to explain why their existence is warranted. I believe the answer to the title of this article comes in three forms:

## 1. Use Case
The Standard Template Library is perhaps _too generic_ for some. C++ is a language that encourages low-level optimization. So when tasked with creating a generic containers library, a balance must be made between functionality and performance. How often will a user need to access the back of a linked list? Storing a back pointer will cost an extra pointer of stack space as well as a small bit of extra computation to maintain it. Is the benefit of the added functionality of maintaining a back pointer so beneficial to those users who need it that it's worth the performance hit to those users who don't? Why are we even worrying about the stack-size of the container? Isn't the heap-size more important?

These are questions that are impossible to answer and those who implement the C++ standard library have done their best guess at assuming the use cases.

However, when the users are those in a certain specialized industry, and heck they may even be your coworkers sitting in the desk next to you, we can start to make some more educated guesses on how these containers might be used.

Following is a table of the byte-sizes of my different containers compared to the stl. This was compiled with MSVC in release mode. Your stl sizes may differ depending on your compiler.

|                      | container | `iterator` |
|:--------------------:|:---------:|:----------:|
| `std::forward_list`  | 8         | 8          |
| `SList`              | 24        | 8          |
| `std::vector`        | 24        | 8          |
| `Array`              | 16        | 8          |
| `std::unordered_map` | 64        | 8          |
| `HashMap`            | 24        | 24         |

So it looks like in my types are about the same size or smaller than the stl's, with one exception. It may not seem like much, but we're talking about the fundamental containers that applications will be built upon. These bytes will add up and could contribute to some serious bloat. Let's see what justifications I have then...

Starting with the singly linked list, the standard decided that they'd only maintain a head pointer. I decided a tail pointer would be nice as well as a `size_t` for a count of the number of elements. In this instance, there's not really any choice but to reimplement the container. If you were to be lazy and try to reuse existing code by composing `SList` of a `std::forward_list` you'd have no way to retrieve, store, and the tail pointer since the nodes are not exposed.

The lists' iterators are both the same size, a pointer to a node. An alternative implementation would be to also carry a pointer to the owner list so that you may verify you're performing operations on the correct container when passing the `iterator` into methods on `SList`. I in fact have this enabled in debug, but for release builds I keep the smaller data type.

Now for the contiguous containers. `std::vector` is 24 bytes because it maintains three pointers: the starting address, the ending address, and the address of the last element. I maintain semantically the same information with a starting address, an element count, and a memory capacity. Originally, the count and capacity were both a `size_t` meaning they would be the same width as pointers and so my `Array` would be the same size as a `std::vector`. However, I _really_ don't forsee anyone putting over 4-billion things into my container, so I made the use-case decision to internally store these integers as `uint32_t`s.

Once again, our iterators are the same size, and only contain a pointer to the address of the element being referenced. My original design called for a pointer to the owning `Array` and an index into that array. This would double the size of the `iterator` but the benefit is bounds checking. I originally had many methods throw exceptions when going out of bounds, and safely capping `operator++` so that it never went beyond the end. I've now traded that functionality for performance. We're still getting _some_ bounds checking. Like `SList::iterator`, in debug mode we still have the owner pointer, so we can assert whenever the iterator is being used wrong. I generally prefer exceptions but it's a bad idea to only throw in debug mode.

Finally, the `HashMap`. Here you can see I have a big advantage over stl at over half the size. My HashMap is comprised of only an `Array<SList>` and a `size_t` element count. `std::unordered_map` gets all of its extra bloat by instantiating its `Hash` and `KeyEqual` functors passed in as predicates. My original design for all of these containers in fact relied heavily on storing `std::function`s like these, which resulted in severely bloating the size of these containers since a single `std::function` can be as much as 64 bytes! In my new design, I don't store these at all. I instead assume that `Hash` and `KeyEqual` are both stateless and trivially default constructable so that I may simply instantiate them on the stack as-needed. This is a **use case** decision as I don't see any reason for anyone using my `HashMap` to pass a `Hash` functor with state.

While I believe I have a great advantage in the size of the container itself, the size of the `iterator` here is a severe blow, and I believe very telling of how `std::unordered_map` is implemented. I decided to be lazy and compose my `HashMap::iterator` out of an `SList::iterator`, `Array::iterator`, and an owner pointer. This makes perfect sense since my `HashMap` is nothing but an `Array` of `SList`s, so I need to be able to iterator through the `Array` and then iterate though each `SList`. So how the heck does the stl get away with only 8 bytes when they use an array of singly linked lists too?

All their linked-lists must be connected. Instead of implementing it as an "array of linked lists" it's "a single linked list with an array of pointers to nodes". So they can still get O(1) random access into a region of the list while at the same time having an easy way to get to the next bucket from the last node of the previous bucket. Fascinating.

As a final note on this topic of "use case", I'd like to mention allocators. All stl containers have a template parameter for an allocator which defines how memory is allocated for that container. Since these containers I'm making are for a game engine, I don't believe allocators are necessary. Memory should be managed by the engine, not the user. Though I do provide something similar for my `Array` and `HashMap`: a "resize strategy". This is a stateless predicate which returns what the new container capacity should be given the current size and capacity. With this, I'm telling users that they don't have control over exactly _how_ memory is allocated, but they do have control over exactly how _much_ and how _often_ memory is allocated.

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

What if you want to remove all elements matching a certain predicate? C# conveniently provides `List<T>.RemoveAll(Predicate<T>)`, so C++ probably has something similar, right? No? Enter the erase-remove idiom:

```c++
const auto it = std::remove(v.begin(), v.end(), t);
v.erase(it, v.end());
```

Now _that_ is ugly. And what the heck? You're calling some generic free function `std::remove` to do it? Which returns an iterator?

This highlights my personal biggest gripe with the stl. C++ is a language of patterns. It is in fact more focused on these patterns than Object Orientation, which can seem completely bizarre to programmers coming from C# or Java. What is undoubtedly the king of these patterns is the iterator pattern. Iterators are extremely powerful, and it's thanks to them that generic methods such as `std::remove` can work on almost any object. So from the perspective of code reuse, free functions like `std::remove` are beautiful. There doesn't have to be a separate `remove` method on every container now. That rocks, but it doesn't make it any easier to use.

I believe that libraries should convenience themselves to the users as much as possible. No novice using C++ is going to discover the erase-remove idiom on their own. They're going to type `myVector.` and scroll through the intellisense popup looking for the `remove` method that isn't there. Then they'll see `erase`, figure it must mean the same thing, and then go to StackOverflow asking why their code doesn't work.

Unreal's `TArray` provides a `Remove` method, and so does my `Array`.

Returning to the topic of allocators, there are some severe annoyances that users might have run into with using mixed allocators. If you have a `std::vector<int, std::allocator<int>>` and a `std::vector<int, MyAllocator<int>>` these are _two different types_ and the _compiler will treat them as such_. Which means that there is no `operator==` between them. Even though they both contain ints, just because those ints might have been allocated differently there's no way to compare them. The user is forced to use `std::equal(v1.begin(), v1.end(), v2.begin(), v2.end())`.

Even though my containers don't use allocators, I'm in the same conundrum. I instead have reserve strategies for my `Array` and this problem can persist for the `KeyEqual` and `Hash` functors of `std::unordered_map`/`HashMap`. But it doesn't have to be this way. I've provided a _templated_ `operator==` so two `Array`s with different reserve strategies:

```c++
template<Concept::ReserveStrategy OtherReserveStrategy>
friend bool operator==(const Array& left, const Array<T, OtherReserveStrategy>& right);
```

It doesn't stop there though. What about copy and move semantics? Indeed, there's no way to (easily) copy-construct a `std::vector<int, std::allocator<int>>` from a `std::vector<int, MyAllocator<int>>`. Templates to the rescue!

```c++
template<Concept::ReserveStrategy OtherReserveStrategy>
Array(const Array<T, OtherReserveStrategy>& other);
```

Notice my use of `Concept::ReserveStrategy`. This is a custom concept I've created which ensures that the user pass in a proper functor:

```c++
template<typename T>
concept ReserveStrategy = requires(T t, size_t size, size_t capacity)
{
	{ t(size, capacity) }->std::convertible_to<size_t>;
};
```

This is just the beginning of the delicious C++20 goodness I'm baking into my library. While the above example of C++20 usage is only really to provide additional safety by preventing the user from passing in some random type that may not even be a functor at all, this next example provides some real ease-of-use:

```c++
template<Concept::RangeOf<T> Range>
Array(const Range& range);
```

A range-based copy-constructor! Many stl containers provide an iterator constructor like the following:

```c++
template<typename It>
std::vector(It first, It last);
```

But why even ask the user to pass in `begin()` and `end()`? Why not just let them pass in the whole thing to be copied? Want to make an Array from a `std::deque`? Be my guest:

```c++
Array<int> a = someDequeOfInts;
// or even reassign it with my assignment operator overload
a = someOtherDequeOfInts;
```

I do still provide iterator ctors though because you might want to construct the container from `begin() + 1` to `end()`.

## 3. Not-Invented-Here Syndrome
We're all guilty of having this at least partially. Some might have it worse than others. I might be one of them. All the examples and reasons I've given above are great and all but if I've got to be honest, this is the real reason I've re-implemented all of these stl containers. Doing it myself has been a fun and rewarding learning experience. I now feel more confident in my usage of C++ and even in the stl containers I've been working so hard to replace.

But there's more to this than just the gratification of doing it yourself. There's a real security in having the source code available to read, modify, and verify accuracy. Sure, whatever implementation of the stl is available to you is likely open source, but it's just as likely _ungodly unreadable_ to the point that it almost doesn't count. Plus if the code is developed in-house, ~~if~~ when problems arise, you can (if you're this type of studio) just wander over to the author's desk for help, which beats putting in a support ticket.

--------------------------------

But does all of the above really warrant "reinventing the wheel"? In my opinion, for the vast majority of use cases, no. The stl was developed as a performant general-use library and they've achieved exactly that. It's got some very strange... C++isms that make it uncomfortable to approach for newcomers, but once you learn the ropes it's not that bad. Whatever benefits you might get from reimplementing the standard library are likely going to be outweighed by the _deep_ time sink of doing so. When you get down to it, time - _human_ work time not _runtime_ - might be the most important factor to consider. A large company who takes on a lot of inexperienced employees might earn back that time spent if their custom library is more programmer friendly and those new hires can learn more easily and become productive quicker.

Then again, I'm pretty inexperienced myself, so forget everything I just said.
