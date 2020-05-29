---
title: Macros Are Your Friend
author: Jarrett Wendt
excerpt: Don't shun it like that person you didn't invite to the party.
thumb: assets/images/WarGamesTicTacToe.gif
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
layout: post
cwd: '../'
---
{% assign customEnginePage = site.pages | where: 'title', 'Making a Game Engine from Scratch' | first %}

<figure>
<img src="{{ page.cwd }}{{ page.thumb }}" alt="WargamesTicTacToe" width="500" style="display: block; margin-left: auto; margin-right: auto;">
<figcaption>don't play WarGames with your precompiler</figcaption>
</figure>

If you've been noticing a trend among each new release of the C++ standard, you're not the only one. It seems to be among the C++ standard's committee's top priorities to kill the precompiler. We've got `constexpr` variables to replace `#define` constants, `constexpr` functions to replace `#define` macro functions, _constexpr-if_ to replace `#if`, modules to replace `#include`, and of course templates to replace a lot of the dirty uses of `#define` in good old C.

But there's something that the precompiler can still do that hasn't been replaced yet: generating incomplete code.

I try to stay away from macros as much as any other modern C++ developer, but there's some tricks you can accomplish with it that can save you a lot of bothersome typing. In this article I'll walk through a few of my favorites I've put together while developing <a href="{{ customEnginePage.url }}" target="_blank">my game engine from scratch</a>.

## Assert with Message
You could argue that throwing exceptions is a modern alternative to `assert` and you might have a point. But `assert`s are still very different and very important to modern application development. Unlike exceptions, `assert`s will not be compiled for release builds, meaning all overhead associated with them is gone. This makes them ideal for bugs you only expect to show up during development. That leaves exceptions for bugs that can appear during release, such as user-input errors.

What's not helpful about `assert`s is what happens when you get one. The entire application crashes, likely ruining any debug information you might've had, and only providing a popup window displaying the line in which the `assert` is on. This could be even less helpful if that line is merely:
```c++
assert(myBool);
```

Wouldn't it be great if you could provide a message with that `assert`? Like providing a string to the constructor of an exception? Not only then would the `assert` provide more information when it rears its ugly head, but its intentions will be more clear within the code.

People have come up with plenty of tricks to make this happen, but my favorite is the good old `&&`:
```c++
#define assertm(exp, msg) assert(exp && msg)
// ...
assertm(ptr != nullptr, "function returned nullptr in Namespace::Class::Method")
```

## Lazy-Rule-Of-6
C++ has an annoying set of rules for what special members are provided by default when. As such, I like to follow the "Rule of 6" which means to _always_ provide a constructor, destructor, move/copy constructor, and move/copy assignment operator. That can get really tedious to write out, especially when you just want to default them all:
```c++
MyClass() = default;
MyClass(const MyClass&) = default;
MyClass(MyClass&&) noexcept = default;
MyClass& operator=(const MyClass&) = default;
MyClass& operator=(MyClass&&) noexcept = default;
~MyClass() = default;
```

After putting up with copy/pasting the above code several times and every time replacing `MyClass` with `MyActualClass` I decided to write a macro to do it for me:
```c++
#define SPECIAL_MEMBERS(Type, D)		\
Type() = D;								\
Type(const Type&) = D;					\
Type(Type&&) noexcept = D;				\
Type& operator=(const Type&) = D;		\
Type& operator=(Type&&) noexcept = D;	\
~Type() = D;
```

Of course, I have to have a separate one for when we want a `virtual` destructor:
```c++
#define SPECIAL_MEMBERS_V(Type, D)		\
Type() = D;								\
Type(const Type&) = D;					\
Type(Type&&) noexcept = D;				\
Type& operator=(const Type&) = D;		\
Type& operator=(Type&&) noexcept = D;	\
virtual ~Type() = D;
```

But Jarrett, why allow passing in a parameter for the value you're setting them to? Wouldn't you want to always `default` them? Not so! While C++ doesn't provide an explicit syntax for defining a static class, one can be achieved by marking all special members for `delete` and only having `static` methods. Thus, we can create a really nifty macro:
```c++
#define STATIC_CLASS(Class)				\
SPECIAL_MEMBERS(Class, delete)
```

This actually produces code that I find more self-documenting than explicitly deleting all special members:
```c++
class MyClassOfUnknownPurpose
{
	// Oh! It's a static class! Well that clears some things up.
	STATIC_CLASS(MyClassOfUnknownPurpose)
};
```

## Comparison Operators
The glorious <a href="https://en.cppreference.com/w/cpp/language/default_comparisons" target="_blank">spaceship operator</a> is almost here. Unfortunately, I haven't gotten it to work fully under MSVC yet. I can define `operator<=>` and call it explicitly, but it doesn't automatically generate the rest of the comparison operators.

So for now, I'm stuck with defining all the comparison operators explicitly. Myself and others probably be in this scenario for years to come as projects slowly upgrade through the C++ versions as they tend to. This macro I'm getting to will be helpful to myself and others who are stuck in a C++17 or earlier project.

If you only define `operator==` and `operator<` you can actually define all other comparison operators based off of it. Here's a macro that does it all for you:
```c++
#define COMPARISONS(Class)								\
bool operator!=(const Class& other) const noexcept		\
{ return !operator==(other); }							\
bool operator<=(const Class& other) const noexcept		\
{ return operator==(other) || operator<(other); }		\
bool operator>(const Class& other) const noexcept		\
{ return operator!=(other) && !operator<=(other); }	\
bool operator>=(const Class& other) const noexcept		\
{ return operator==(other) || operator>(other); }
```

## const_iterators From Non-Const iterators
Now _this_ is the real reason I wanted to write this article. The macros I've shown so far save maybe 6-10 lines of code, but this has saved me hundreds.

I got really tired of writing out an `iterator` and then doing everything all over again for a `cosnt_iterator`. Then I realized: a `const_iterator` can be completely defined by a non-const `iterator` by simply making it a wrapper to the non-const version:
```c++
#define CONST_FORWARD_ITERATOR(CIT, IT)											\
private:																			\
	IT it;																			\
public:																				\
	using iterator_category = typename IT::iterator_category;						\
	using value_type = typename IT::value_type;									\
	using difference_type = typename IT::difference_type;							\
	using pointer = const typename IT::pointer;									\
	using reference = const typename IT::reference;								\
	CIT(const IT it) noexcept : it(it) {}											\
	SPECIAL_MEMBERS(CIT, default)													\
	CIT operator++(int) noexcept { return it.operator++(0); }						\
	CIT& operator++() noexcept { it.operator++(); return *this; }					\
	reference operator*() const { return it.operator*(); }						\
	pointer operator->() const { return it.operator->(); }						\
	bool operator==(const CIT other) const noexcept { return it == other.it; }	\
	bool operator!=(const CIT other) const noexcept { return it != other.it; }
```

And here it is in action:
```c++
struct const_iterator final
{
	CONST_FORWARD_ITERATOR(const_iterator, iterator)
}
```

Kinda strange to see an iterator one-liner, isn't it? I've created versions of this macro for not just forward but also bidirectional and random-access iterators too. The higher-tiered iterator categories simply invoke the macros of the next lower-tiered iterator category plus defining whatever members are unique to that particular iterator category.

## Self-Writing Random-Access Iterators
Similar to how all comparison operators can be implemented based off of just two operators, all random-access iterator operators can be defined from just two operators: `operator+` and `operator*` (plus we'll be using the `COMPARISONS` macro trick too).

```c++
#define RANDOM_ITER_OPS(IT)															\
IT operator++(int) noexcept { const auto ret = *this; operator++(); return ret; }		\
IT& operator++() noexcept { *this = operator+(1); return *this; }						\
IT operator--(int) noexcept { const auto ret = *this; operator--(); return ret; }		\
IT& operator--() noexcept { *this = operator-(1); return *this; }						\
friend IT operator+(const difference_type i, const IT it) { return it.operator+(i); }	\
IT operator-(const difference_type i) const noexcept { return operator+(-i); }		\
IT& operator+=(const difference_type i) noexcept { return *this = operator+(i); }		\
IT& operator-=(const difference_type i) noexcept { return *this = operator-(i); }		\
pointer operator->() const { return &operator*(); }									\
reference operator[](const difference_type i) const { return *(*this + i); }			\
COMPARISONS(IT)
```

The only random-access iterator operator that can't be defined here but also can't be inferred from `operator+` and `operator*` is the one with the following signature:
```c++
friend difference_type operator-(const iterator left, const iterator right);
```

All-in-all, that leaves a total of 5 necessary operators to implement for the beefiest of iterators, and the rest is taken care of by _macro-magic_!

## All of the begin/end!
Why stop at just defining the iterators? Those `begin`/`end` methods get tedious too, especially since most of them just call the others. I think you can see where this is going...
```c++
#define BEGIN_END(IT, CIT, CLASS)												\
CIT begin() const noexcept { return CIT(const_cast<CLASS*>(this)->begin()); }	\
CIT cbegin() const noexcept { return begin(); }								\
CIT end() const noexcept { return CIT(const_cast<CLASS*>(this)->end()); }		\
CIT cend() const noexcept { return end(); }
```

We can do it for reverse-iterators too:
```c++
#define MAKE_REVERSE_BEGIN_END													\
auto rbegin() noexcept { return std::make_reverse_iterator(end()); }			\
auto rbegin() const noexcept { return std::make_reverse_iterator(cend()); }	\
auto crbegin() const noexcept { return std::make_reverse_iterator(cend()); }	\
auto rend() noexcept { return std::make_reverse_iterator(begin()); }			\
auto rend() const noexcept { return std::make_reverse_iterator(cbegin()); }	\
auto crend() const noexcept { return std::make_reverse_iterator(cbegin()); }
```

Whew, now I can rest my weary fingers...

---

Many people treat the preprocessor like that friend you're not sure who invited to the party. You can't ask them to leave, but you don't want them here. You interact with them occasionally, when it's socially required of you to, but otherwise you try to stay as far away from them as possible.

That's not a very productive situation. Instead of shunning the preprocessor, welcome it. Get it involved in the party, ask it how it's doing. Who knows, it might be the beginning of a beautiful friendship...

Until the C++ standard's committee comes up with something that can do all the macros I just listed above. At which point the party's over and the preprocessor is passed out on the couch, vastly overstaying its welcome.
