---
title: Reflection in C++
author: Jarrett Wendt
excerpt: A little bit goes a long way.
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
thumb: assets/images/darth_kermit.png
layout: post
cwd: '../'
---
{% assign vvv = site.custom_engine | where: 'title', 'Visiting Variant Vectors' | first %}
{% assign reinventingTheWheel = site.custom_engine | where: 'title', 'Reinventing the Wheel' | first %}

By virtue of writing code, we're extending the language given to us. Whenever I extend C++, the language I most often look to for inspiration is C#. C# has a lot of features I wish C++ had such as properties, extension methods, and discards. However, perhaps the most powerful feature of C# is its extensive native runtime reflection system.

Reflection feels like programming magic. To be able to take a string, _any_ string, even one a user typed in, and retrieve some info off of an object, perhaps a method with that name and invoke it, is like witchcraft.

So how can we get something like this in C++?

The first question we need to answer before that is to analyze exactly how much reflection we want for our current project. For your current needs, you may not actually need _run-time_ reflection. Understand, C++ already has an extension compile-time reflection system already. Through type-traits and `constexpr`s, we can achieve <a href="{{ vvv.url }}" target="_blank">some pretty ludicrous things at compile-time</a>.

That said, compile-time reflection may not be enough for your use case, and indeed modern game engines require some kind of run-time reflection system. Unity of course has native reflection in C# and Unreal has a custom build tool that pre-pre-parses the code before even the precompiler runs.

## Markup

In <a href="{{ vvv.url }}" target="_blank">my article on variant data types</a> I eluded to the type of reflection I desired for my engine. Let me provide a more complete example:

```c++
class MyClass
{
public:
	[[Attribute]]
	int myInt;
};

// ...

MyClass c;
Datum d = c.Attribute("myInt");
int& i = d.Get<int>();
assert(&i == &c.myInt);
```

This should look very similar to what's possible in Unreal with their `UPROPERTY` macro. That macro actually doesn't do anything. It's just a special piece of markup in the code for the Unreal Build Tool (UBT) to look for. When the UBT sees a `UPROPERTY` it creates reflection info for it and puts it inside of `MyClass.generated.h`.

The `[[]]` syntax is an <a href="https://en.cppreference.com/w/cpp/language/attributes" target="_blank">attribute specifier</a>. The C++ standard defines a few attributes that all compilers should recognize, but any unrecognized attribute _will_ be ignored by the compiler. This means we don't get any syntax errors when we type `[[ whatever we want! ]]` between the double-brackets. This was added to the language in C++11 for the exact reason `UPROPERTY` exists: to mark-up the code so that compilers (or pre-compilers) can do special things with it.

To be able to do anything with a custom attribute, we must write our own pre-parser. I was not particularly concerned with speed here. It doesn't bother me if the pre-compiling phase is slow or inefficient. If you ask me, if there's any time for an application to be slow or inefficient, it's at compile-time. I also knew that this tool would involve a _lot_ of string parsing, so a language with a lot of helpful libraries for doing that would be excellent. Thus, the language I chose to write my pre-parser was Python.

My pre-parser is extremely rudimentary. It simply loops through the source files looking for "`[[Attribute]]`" and it grabs information about the following line of code when it finds one. Ideally, the pre-parser would utilize an _actual_ compiler API to help it along. This would allow it to be smarter in cases such as:
- When "`[[Attribute]]`" exists in a comment.
- When "`[[Attribute]]`" is placed in an illegal location.
- Getting the name of the class the "`[[Attribute]]`" occurs in (my parser just assumes the name of the file).

Unfortunately, C++ is a notoriously difficult language the parse. I wasn't able to find many compiler API libraries for Python and those that I did find didn't have any support for attributes since that _is_ a kind of niche syntax in the language. So improving the pre-parser remains one of the biggest TODOs in my engine. I'm not totally tied to _only_ using Python for my pre-parser, so perhaps when I revisit this I'll try another language.

## Attributed

The next piece of the puzzle is retrieving the reflection information the pre-parser generated from C++ at run-time. All classes that use the `[[Attribute]]` syntax must inherit from a common base class called `Attributed`. An `Attributed` stores a `HashMap` of strings to external `Datum`s. The reason the `Datum`s are external is because they reference the memory of the `[[Attribute]]` in the derived class, not some heap-allocation that the `Datum` owns.

So whenever we say
```c++
myAttributed.Attribute("myInt");
```
We're really just indexing into the `HashMap` and retrieving the value. But how does that `HashMap` get built up?

On construction, an `Attributed` will automatically populate the `HashMap` it does this using the information from the pre-parser. In my engine, I have a static `HashMap` with reflection information about all `[[Attribute]]`s. The information includes:
- The name of the variable.
- The `offsetof` that variable.
- The type of that variable as an enumeration.

The pre-parser's output is a `.cpp` file that _is_ the instantiation of that static `HashMap`. Here's a simplified example of what `Attributes.generated.cpp` could look like:

```c++
#include "full/path/to/MyAttributed.h"

const HashMap<RTTI::IDType, AttributeInfo> registry =
{
	MyAttributed::typeID,
	{
		"myInt",
		offsetof(MyAttributed, MyAttributed::myInt),
		Datum::TypeOf<int>
	}
};
```

With this registry of reflection information, we can get the addresses of all `[[Attribute]]`s on the derived type from the base ctor of `Attributed`:
```c++
protected:
Attributed::Attributed(const IDType derived)
{
	for (auto [name, offset, type] : registry[derived])
	{
		void* addr = reinterpret_cast<uint8_t*>(this) + byteOffset;
		attributes[name] = Datum(type, addr);
	}
}
```

`Attributed` has no default constructor. That means any type that derives from `Attributed` must explicitly invoke this `protected` constructor, passing along their run-time-type-information ID. The reason for this is because of how C++ handles `virtual` methods within constructors. Any invocation of a `virtual` method within a constructor will be statically resolved. That-is, resolved at compile time. That-is, we _won't_ get `virtual` polymorphic behavior. Rather, we will _only_ call the version of that method for the type of the constructor we're currently in. So within the constructor of `Attributed` we have absolutely no way to determine what derived type we are. Therefore, we must pass along the derived type explicitly. Here's what that looks like in a class:

```c++
class MyAttributed : public Attributed
{
	[[Attribute]]
	int myInt;

protected:
	// to be invoked by anything that inherits from MyAttributed
	explicit MyAttributed(RTTI::IDType derived) : Attributed(derived) {}
public:
	MyAttributed() : MyAttributed(typeID) {}
};
```

That can be a little bit of an annoyance on the user to remember to pass along the derived type ID with a protected ctor. So I wrote a macro which does it automatically (if you're okay with default ctors):
```c++
#define ATTRIBUTED_SPECIAL_MEMBERS(DerivedType, BaseType, D)			\
protected:																\
	explicit DerivedType(RTTI::IDType derived) : BaseType(derived) {}	\
public:																	\
	DerivedType() : BaseType(typeID) {}								\
	DerivedType(const DerivedType& other) = D;							\
	DerivedType(DerivedType&& other) noexcept = D;						\
	DerivedType& operator=(const DerivedType& other) = D;				\
	DerivedType& operator=(DerivedType&& other) noexcept = D;			\
	virtual ~DerivedType() = D;
```

---

A clear difference here between my pre-parser and Unreal's is that it only spits out _one_ generated file whereas Unreal creates at least one for every class being reflected. Unreal is also using their build tool to do a lot more than my system is currently. As my engine grows and my pre-parser gets more sophisticated, I'll likely end up splitting the reflection information up into multiple files.
