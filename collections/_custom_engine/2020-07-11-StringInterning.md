---
title: String Interning
author: Jarrett Wendt
excerpt: More than just saving space.
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
layout: post
cwd: '../'
---

{% assign vvv = site.custom_engine | where: 'title', 'Visiting Variant Vectors' | first %}
{% assign needlesslyCleanSyntax = site.custom_engine | where: 'title', 'Needlessly Complex Code to Achieve Needlessly Clean Syntax' | first %}

## What is Interning?
Just a quick preface in case you don't know and you don't feel like visiting <a href="https://en.wikipedia.org/wiki/String_interning" target="_blank">Wikipedia</a>. String interning is when all strings are pooled such that there's no duplicates. For example:
```c++
String a = "hello";
String b = "hello";
```
These two strings would not take up 12 bytes (including null terminators), but rather 6 bytes. The declaration of `a` puts `"hello"` into a pool and `b` does not create a duplicate but rather references the original string created by `a`.

This is already a feature in C/C++ granted by most compilers. You can see for yourself with a short program like this:
```c
#include <stdio.h>

int main()
{
	printf("%p\n", "hello");
	printf("%p\n", "hello");

	return 0;
}
```

You should see that these two separate `"hello"` literals are indeed the same pointer. Unfortunately, that's the limit of string interning in pure C/C++. It's only for string literals.

Meanwhile, many higher level languages such as C# implement runtime string interning. They may even go as far as to intern substrings such as if you have a string `"hello, world!"` then a substring such as `"hello"` costs no extra memory. 

## Motivation
Besides the obvious memory savings, I had another motivation. I'm using strings everywhere in my engine thus far. Primarily as keys into hash tables. Strings make great hash keys because they can be human readable and are easily hashable. However, I'd prefer not to re-hash these strings if I can help it. So what if we could memoize these hash values such that they're only ever calculated _once_?

## What about Tries?
Whenever I'm faced a problem with strings my default solution is always to use a trie. Unfortunately, it rarely turns out to be the best solution. One of the biggest disadvantages of tries is all the pointer indirection. For something as fundamental as a string I'd rather not have to dereference a pointer for every character, that's going to guarantee some cache thrashing.

Plus, no matter what, we're going to need some way to convert our interned string into some common string-type denominator such as `std::string` or `const char*`. There's no way to easily retrieve a `const char*` out of a trie, it has to be built up by recursive descent. So if we have to re-build the string every time we're going to use it then there's not much point in our optimization. For an interned string type, I demand $$O(1)$$ direct access, not $$O(k)$$ where $$k$$ is the length of the string.

A trie would certainly have its merits here though. For instance, we would instantly get substring interning. Or at least prefix interning. If we have `"hello, world!"` in our trie then `"hello"` would cost no extra memory but `"world"` would.

## Implementation
So rather than a trie, the next best option would be a hash table. This actually works out great because we were planning to memoize the hash value anyway so we can recycle that.

So first of all we need our internal type that a single string will be interned with:
```c++
struct Intern
{
	const std::string string;
	const hash_t hash;

	explicit Intern(const std::string& string) :
		string(string),
		hash(std::hash<std::string>{}(string)) {}
}
```
As you can see, we store our strings as a good old `std::string` and on construction we compute its hash value with `std::hash`.

We can then store these inside a `std::unordered_set`. However, we need to think about how we're storing them inside the set so that our wrapper which references this interned value doesn't become dangling. So we could heap-allocate our `Intern`s and store them as raw pointers or better yet we could use `std::shared_ptr` as that would give us a reference count to potentially remove `Intern`s when they are no longer in use.

We're going to be using this `shared_ptr` a lot so I'm going to make an alias to save a little typing:
```c++
using SharedIntern = std::shared_ptr<Intern>;
```

Now when using `std::unordered_set` we need to create a custom hash functor and comparison functor or else we will get the default behavior with `std::shared_ptr` which is to just use the address.

```c++
struct Hash
{
	constexpr hash_t operator()(const SharedIntern& i) const
	{
		return i->hash;
	}
};

struct KeyEqual
{
	bool operator()(const SharedIntern& a, const SharedIntern& b) const
	{
		return a->string == b->string;
	}
};

static inline std::unordered_set<SharedIntern, Hash, KeyEqual> set{};
```

Finally, our interned string type is simply a reference to one of these `Intern`s:
```c++
class String
{
	SharedIntern intern;

public:
	String() noexcept :
		String("") {}

	String(const char* string) noexcept :
		String(std::string(string)) {}

	String(const std::string& string) noexcept:
		intern(*set.emplace(std::make_shared<Intern>(string)).first) {}

	String(const String& other) noexcept = default;
	String(String&& other) noexcept = default;

	constexpr operator const std::string&() const noexcept
	{
		return intern->string;
	}
};
```
You can see that on construction we make our `Intern` and hash it. The behavior of `unordered_set::emplace` is such that it will only insert if the value doesn't already exist, which is exactly what we want. So no matter what we get back a valid `SharedIntern` which we can dereference whenever we want to get the actual string, such as in our implicit conversion operator for casting to a `const std::string&`.

```c++
String::~String()
{
	if (intern && intern.use_count() <= 2)
	{
		set.erase(intern);
	}
}
```
On destruction we can check the current reference count and decide on whether or not we should erase this `Intern` from the set. Assuming this is the last reference to our `Intern` then the reference count should be exactly 2:
- 1 reference is always held by the set.
- We also have our own reference.

It should not be possible to have a reference count less than 2 and indeed in my own unit tests I never found such a case. But as a sanity check I like to use `operator<=` here.

We only want to bother erasing if the `shared_ptr` is still valid. It could have gone bad if this destructor is being called after the object was moved.

```c++
String& String::operator=(const String& other)
{
	if (this != &other)
	{
		this->~String();
		intern = other.intern;
	}
	return *this;
}

String& String::operator=(String&& other) noexcept
{
	if (this != &other)
	{
		this->~String();
		intern = std::move(other.intern);
	}
	return *this;
}
```
In both assignment operators we need to explicitly invoke the destructor. Not necessarily because we need to "destroy this" but because the destructor happens to contain the exact logic we need to run, which is to potentially erase this intern from the set in the event that this object had the last reference to that string.

```c++
String operator""_s(const char* str, size_t length) noexcept
{
	return String(std::string(str, length));
}
```
I also of course provided a literal operator for convenience. Not shown are all the utility methods I wrote, most of which are just wrappers to `std::string`'s methods.

## Pros and Cons
The biggest bummer of my implementation over using a bog standard `std::string` is that the constructor isn't and can't be `constexpr`.

Here's an example that exacerbates the issue:
```c++
for (int i = 0; i < 100; i++)
{
	std::cout << "hello, world!"_s << std::endl;
}
```
This code will create, destroy, hash, and do a set insertion 100 times. Meanwhile if this were a `std::string` or even a `const char*` then it would have been calculated once by the compiler and likely even compile-time interned.

The benefit of my implementation is run-time interning and $$O(1)$$ (as opposed to `std::string`'s $$O(n)$$ deep copy). Ideally, I'd like to come up with some sort of implementation that combines the benefits of `std::string`'s `constexpr` interning for literals and my run-time interning.

If you've read any of my <a href="{{ vvv.url }}" target="_blank">previous</a> <a href="{{ needlesslyCleanSyntax.url }}" target="_blank">blog posts</a> you'll know that I _love_ letting the compiler do the work for me. So I'm going to keep looking into this and will update this page if I figure something out.

Otherwise, I'm extremely happy with how this turned out. The overhead per-string is minimal:
- A reference count per-string
- A `std::unordered_set` node per-string
- An $$ O(k) $$ where $$ k $$ is the length of the string hash computation on initial construction.
- A `std::unordered_set` insertion on initial construction.

Which I believe is minimal compared to the benefits:
- $$ O(1) $$ copies.
- $$ O(1) $$ hashing.
- No duplicate strings.
