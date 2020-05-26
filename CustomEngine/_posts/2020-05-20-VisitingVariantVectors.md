---
layout: customEngine
author: Jarrett Wendt
title: Visiting Variant Vectors
category: customEngine
---

The title isn't 100% accurate but I liked the alliteration. We'll actually be visiting `Array`s instead of `std::vector`s. See my last post for more on my `Array` type.

Every game engine has some sort of a Variant type. Unreal has [FVariant](https://docs.unrealengine.com/en-US/API/Runtime/Core/Misc/FVariant/index.html), Godot has their own [Variant](https://docs.godotengine.org/en/stable/development/cpp/variant_class.html), in Unity you have the option to use C#'s `dynamic` type.

So what are these used for? Well, data serialization mainly. Particularly, serialization between the code and the editor. Consider Unreal's `UPROPERTY` types (or Unity's `[SerializeField]` if you prefer). How does the engine take some unknown data type and display it in the editor?

The first hint is to recognize that not every type is supported. Unity's `[SerializeField]` only works with types marked `[System.Serializable]`, and `UPROPERTY` only works with Unreal's types such as their `typedef`ed primitives, FStructs, and UClasses.

The next hint is to recognize that the user isn't actually creating any code that interfaces directly with displaying this data in the editor. All the user does is add some markup to the type and everything else is taken care of. Clearly, something must be parsing the file, parsing our line of code declaring a type, and adding it to the editor. I'll talk more specifically on these parsing techniques in a future article, but for now I'll just call this the "Markup Parser".

The final hint is to recognize that programmers are lazy. We want to avoid as much duplicate code as possible, which means that we don't have separate logic for when the Markup Parser see's an `int` vs a `float` vs a pointer, etc. So all of these types must be getting unified into a single type.

How you'll see this accomplished in most implementations is with a discriminated union:
```c++
union Union
{
	bool b,
	int i,
	float f,
	std::string s,
	void* p
};
```
Often paired with an enumeration:
```c++
enum class Types
{
	Bool,
	Int,
	Float,
	String,
	Pointer
};
```
Thus, a data type that can be any of these would be comprised of this union and an enum keeping track of which type is currently stored:
```c++
class Variant
{
	Union value;
	Type type;
};
```

That's the basics, but there's some serious headaches with how this would be fully implemented. Let's start with the first question you might be asking: How do you get a value out of this class? The most naive approach would be to have a method like the following:
```c++
bool& Variant::GetBool()
{
	return value.b;
}
```
Hopefully you can already see why that's a bad idea. You'd have to make a method like that for each of the types. Similarly, for every method in the class, you'd have to have a different variation for every type you're supporting. On top of that, let's say later down the line you decide you want to add a new type to the union, well you've got to refactor the entire class to add another variation for every method. What a pain!

So how can we make our lives easier? When faced with any generic programming problem in C++, the first answer that should come to mind is templates. Using templates, our method could look more like this:
```c++
template<typename T>
T& Variant::Get()
{
	switch (T)
	{
		case bool: return value.b;
		case int: return value.i;
		case float: return value.f;
		case std::string: return value.s;
		case void*: return value.p;
	}
}
```
Unfortunately, that's not valid syntax in C++. You can't switch over a type. There is an alternative though: _constexpr if_.

> Why did they decide to call it _constexpr if_ when the syntax is `if constexpr`? The world may never know...

What's potentially the most powerful element of C++ is everything that it can accomplish at compile-time. And there's some _seriously_ crazy things you can do at compile time, including conditionally compiling code without using nasty `#ifdef` preprocessor statements. In the olden times, the only way we could conditionally compile code in C++ was by using SFINAE, which had such an ugly syntax I won't even give you an example here. In C++17, we got _constexpr if_ which is the easiest way to add some compile-time goodness to your code. Here's how our method could look using it:
```c++
template<typename T>
T& Variant::Get()
{
	if constexpr (std::is_same<T, bool>::value)
	{
		return value.b;
	}
	else if constexpr (std::is_same<T, int>::value)
	{
		return value.i;
	}
	else if constexpr (std::is_same<T, float>::value)
	{
		return value.f;
	}
	else if constexpr (std::is_same<T, std::string>::value)
	{
		return value.s;
	}
	else if constexpr (std::is_same<T, void*>::value)
	{
		return value.p;
	}
	else
	{
		static_assert(false, "unexpected type");
	}
}
```
[`std::is_same`](https://en.cppreference.com/w/cpp/types/is_same) is an extremely helpful constant expression type trait utility provided by the standard library. Compile-time evaluated traits like this are the bread and butter of powerful compile-time code. You'll also see `static_assert` which is a statement that can force compilation to fail if the compiler ever evaluates it to be `false`. Using _constexpr else-if_ statements, only one of these statements will ever compile. So in the method `Variant::Get<int>` the code for `bool` does not exist at all. That's awesome!

This is a great step in the right direction, but still not really reducing the amount of work we'll have to do. We've just transformed the issue of having to duplicate every method into having to add an extra _else-if_ statement in every method. Conceptually, we could move the logic of all these _constexpr else-if_ statements into their own method, creating only one point of contention where code needs to be refactored if a new type is introduced. But there's a better way...

Enter `std::variant`. This is a type introduced in C++17 which internally does exactly everything we've been talking about. With it, we could implement our type like so:
```c++
// std::variant is not permitted to hold references or pointers, which is a big sad :(
using Variant = std::variant<bool, int, float, std::string>;
```
And our method would instead be the following:
```c++
// instantiate a Variant as a bool with the value false
Variant variant = false;
// will return our bool value of false
std::get<bool>(variant);
// will throw an exception since variant isn't currently storing an int
std::get<int>(variant);
// variant now holds a float!
variant = 1.5f;
```

This is what we're going to be building our final variant data type out of. So with the foundation in place, what do we want our of this type? It's absolutely essential that this type be able to store arrays. But if we can't put pointers in `std::variant` how are we going to do it? Well, what if the entire variant itself were an array?
```c++
using Variant = Array<std::variant<bool, int, float, std::string>>
```
Then if we're only storing scalars we can just treat it as an `Array` of size 1. Problem solved.

But there's a pretty big issue with this implementation: massive memory bloat. The `sizeof` a `std::variant` is the `sizeof` its largest type (plus the width of a `size_t` which is internally used to keep track of which type is currently stored). In this case, that'd be a `std::string`, which is 40 bytes on an x64 system. Yikes! If we were storing an `Array<bool>` that'd be 39 bytes wasted per element. So what's the fix?

Finally, I shall present the first few lines of the actual data type I've created...

## VariantArray

```c++
template<typename... Ts>
class VariantArray
{
	std::variant<Array<Ts>...> variants;
};

class Datum final : public VariantArray<bool, int, float, std::string, std::shared_ptr<RTTI>> {};
```

`VariantArray` is a general purpose data type that supports arrays of variant types with memory as packed as possible. `Datum` is the specific use of `VariantArray` which my engine will be using. `RTTI` is a base type that all objects in the engine will inherit from, and we store `std::shared_ptr`s of them instead of raw pointers for safety.

The parameter pack expansion ends up creating a type like the following:
```c++
std::variant<Array<bool>, Array<int>, Array<float>, Array<std::string>, Array<std::shared_ptr<RTTI>>>
```
Beautiful. That's exactly what we want, since an `Array<T>` is always the same size (16 bytes), no matter the `T`. Since `VariantArray` has no other data members and `std::variant` stores a `size_t` to keep track of the current type stored, `sizeof(VariantArray)` ends up being 24 bytes due to padding, no matter the `Ts`.

`VariantArray` provides wrappers to all of `Array`s methods (which has every method of `std::vector` plus some). Here's an example of how `EmplaceBack` works:

```c++
template<typename T, typename... Args>
T& EmplaceBack(Args&&... args)
{
	static_assert(Util::IsConvertible<T, Ts...>);
	if (!IsType<T>())
	{
		if (IsEmpty())
		{
			SetType<T>();
		}
		else
		{
			throw InvalidTypeException();
		}
	}
	return std::get<Util::BestMatch<T, Ts...>>(variants).EmplaceBack(std::forward<Args>(args)...);
}
```

I'm going to gloss over the helper methods `IsType`, `IsEmpty`, and `SetType` because they do exactly what you'd expect. What's much more interesting are the utilities `IsConvertible` and `BestMatch`. These are compile-time constant expressions just like `std::is_same`. Let's start with the first one...

```c++
template<typename T, typename... Ts>
constexpr bool IsConvertible() noexcept
{
	if constexpr ((std::is_convertible_v<T, Ts> || ...))
	{
		return true;
	}
	else
	{
		return false;
	}
}
```

There's many ways this could have been implemented. It could have used the SFINAE pattern similar to how `std::is_same` is done, but I find _constexpr if_ much easier to read. So what's going on here? Well, first I'd like to point out that all of the standard library's constant expression type traits have a `_v` version that is just a shortcut to saying `::value`. So the following lines are equivalent:
```c++
std::is_convertible<T, Ts>::value
std::is_convertible_v<T, Ts>
```
What `std::is_convertible` does is test for if the two types are implicitly convertible. For example, `float` and `double` are implicitly convertible. If you're like me, you've probably set up your environment to give a warning for when you're narrowing types like this, but an implicit conversion does exist.

The `...` at the end of the _constexpr if_ statement creates a C++17 fold-expression. This expands to
```c++
if constexpr (std::is_convertible_v<T, T1> || std::is_convertible_v<T, T2> || std::is_convertible_v<T, T3> || /* etc. */)
```
So in total, my `IsConvertible` returns true if `T` is convertible to any of `Ts`. And it does so at compile-time. Nice.

The reason we need this is to prevent users from calling `Datum` methods with unsupported types, while also supporting more types than are explicitly listed. For example, though `double` is not listed as one of `Datum`'s types, the user can still put `double`s into a `Datum`. They'll of course be converted to and stored as `float`s internally, but hey, if you're too lazy to write that `f` to make a `float` literal then this is great.

A much better use case of this is for `RTTI`, which is a base class that all objects in this engine will inherit from. Let's say we have a class `Foo` that inherits from `RTTI`. I can't emphasize this enough: `std::shared_ptr<Foo>` and `std::shared_ptr<RTTI>` are **_different types_**. A lot of C++'s template parameter matching relies on types being exact, which can be a huge pain when dealing with inheritance. Luckily, `std::shared_ptr<RTTI>` and `std::shared_ptr<Foo>` are implicitly convertible with no slicing, and thanks to `IsConvertible` this conversion is supported natively by `VariantArray`.

The next piece of the puzzle is `BestMatch`. Given that `IsConvertible` is true, `BestMatch` will return the type from `Ts` that is most similar to `T`.
- If `T` itself is in `Ts` then it returns T.
- If `T` is nothrow convertible (as in there's an implicit conversion, but it's not marked `noexcept`) to one of `Ts`, it returns that type in `Ts`.
- If `T` is convertible to one of `Ts` it returns that type in `Ts`.
- If `T` is not convertible to any of `Ts` the behaviour is undefined. Ideally, it should `static_assert` but I was getting some errors with that.

So, if our `T` is `std::shared_ptr<Foo>`, but our closest match is `std::shared_ptr<RTTI>`, then that's the `Array` we're going to get.

## value_type

It's good practice to provide some type traits to any container you make in C++. They're often required for some stl algorithms to work. The common ones are:
- `size_type`
- `difference_type`
- `reference`
- `pointer`
- `value_type`

And then you should use these when defining your methods. For example, don't define your `operator[]` like this:
```c++
T& operator[](size_t i);
```
instead do it like this:
```c++
reference operator[](size_type i);
```
That way if you ever decide to change your `size_type` from a `size_t` to, let's say, a `uint32_t`, it's as simple as changing an alias.

Speaking of `operator[]` how the heck is this going to work? It doesn't have a template parameter, so how can we call `std::get`? Furthermore, how would an iterator's `operator*` work? It's absolutely essential that `VariantArray` have an iterator since that's one of the most powerful patterns in C++. The answer is, we can't. At least not with `std::get`. What we have to do is defer the call to `std::get` until the last possible moment when the type is known. I've done this by creating a custom `value_type` that is in fact a wrapper to a reference of an element of a `VariantArray`. It's kind of like an iterator without the ability to iterate.

```c++
struct value_type final
{
private:
	size_type index;
	VariantArray& owner;
public:
	template<typename T>
	value_type& operator=(const T& t);

	template<typename T>
	operator T&();

	template<typename T>
	operator const T&() const;

	bool operator==(value_type other) const;
};
```

So let's see what we've got here. Its data is only and index and an owner. Notice that the owner is a reference rather than a pointer. By deciding to do this, I assert that a `value_type` must always have an owner, and it will never be owned by a different object.

There's a templated `operator=`. This is what allows lines like the following to be possible:
```c++
myVariantArray[i] = 6;
```
It infers `6` to be an `int` and will call the corresponding `std::get`. <ins>At compile time</ins>.

Where most of the magic happens is the templated implicit conversion operators. They can be used like so:
```c++
int& myInt = static_cast<int&>(myVariantArray[i]);
```
A `static_cast` is a bit icky but it's what we're stuck with. However, this also completes the last piece of the puzzle to using range-based for-loops:
```c++
for (int& i : myVariantArray)
{
	// ...
}
```
The range-based for-loop will call `VariantArray::begin()` and `VariantArray::end()` to get a `VariantArray::iterator`, call `VariantArray::iterator::operator*` every iteration, which returns a `VariantArray::value_type`, and automatically calls `VariantArray::value_type::operator int&()`. Beautiful.

Finally, we have `operator==`. Notice I pass in another `value_type` by copy here. I'm using the same philosophy as the standard library does with iterators: they're small, so a copy isn't a big deal. Ideally, we should be able to call
```c++
myVariantArray[i] == myVariantArray[j]
```
without having to `static_cast` like this:
```c++
static_cast<int&>(myVariantArray[i]) == static_cast<int&>(myVariantArray[j])
```
because that just looks awful. But how? There's no template parameter here, so how can we call `std::get`? The answer is, we can't. So we have to do something else...

## std::visit

The visitor pattern is often criticized about in C++ circles, but in this implementation it didn't turn out so bad. Let's take a look at how we would use it in `VariantArray`:

```c++
template<typename Callable>
inline constexpr auto VariantArray<Ts...>::Visit(Callable&& callable)
{
	return std::visit(std::forward<Callable>(callable), variants);
}

myVariantArray.Visit([](const auto& array)
{
	for (const auto& a : array)
	{
		std::cout << a << std::endl;
	}
});
```

`std::visit` invokes a callable type on the currently stored type of `variants`. The callable must support every different type in `variants`. Since we're using the `auto` keyword in our lambda, C++ generates a lambda for every type. This is only convenient in this case because we can use `auto`. If this were some other situation in which we were using `std::variant` and we couldn't use `auto` it would be quite exhaustive to write a lambda for every type, and this is where most of the pain with the visitor pattern comes from.

`std::visit` can be invoked on multiple variants too. This is how we use it in `value_type::operator==`...

```c++
bool value_type::operator==(const value_type other) const noexcept
{
	return std::visit([i = index, j = other.index](const auto& a, const auto& b)
	{
		if constexpr (std::is_same_v<AVal, BVal>)
		{
			return a[i] == b[j];
		}
		else
		{
			return false;
		}
	}, owner.variants, other.owner.variants);
}
```

The complexity of passing multiple variants makes the visitor pattern even more complicated. If we couldn't use auto, we would have to write a different lambda for not just every type, but every _combination_ of types. That's a lambda for (`int`, `int`), (`int`, `float`), (`float`, `int`), etc. Yuck.

But we can use auto. And it is good. Notice the _constexpr if_ within the lambda. If the variants have different types, we just always return `false`. If they're the same type we call their `operator==`.

But what if we have a `VariantArray<float, double>`? Would we want to be able to always compare them, even if they're different types? This is a valid point of debate, but in my opinion if there's a possible `operator==` that we can call, why not call it?

```c++
bool value_type::operator==(const value_type other) const noexcept
{
	return std::visit([i = index, j = other.index](const auto& a, const auto& b)
	{
		using A = decltype(a);
		using B = decltype(b);
		using AVal = decltype(a[i]);
		using BVal = decltype(b[j]);
		constexpr bool aComparable = std::equality_comparable<AVal>;
		constexpr bool bComparable = std::equality_comparable<BVal>;

		if constexpr (std::equality_comparable_with<AVal, BVal>)
		{
			return a[i] == b[j];
		}
		// The order here is to make sure we convert with the following priorities:
		// 1.	nothrow conversions have priority over throwing conversions
		// 2.	convert right before converting left
		else if constexpr (aComparable && std::is_nothrow_convertible_v<B, A>)
		{
			return a[i] == static_cast<const A&>(b[i]);
		}
		else if constexpr (bComparable && std::is_nothrow_convertible_v<A, B>)
		{
			return static_cast<const B&>(a[i]) == b[i];
		}
		else if constexpr (aComparable && std::is_convertible_v<B, A>)
		{
			return a[i] == static_cast<const A&>(b[i]);
		}
		else if constexpr (bComparable && std::is_convertible_v<A, B>)
		{
			return static_cast<const B&>(a[i]) == b[i];
		}
		else
		{
			return false;
		}
	}, owner.variants, other.owner.variants);
}
```

Whew that's a lot to take in for a simple `operator==`. Let's start from the top of the lambda.

`decltype` is a keyword that's been around since `C++11`. It's for getting the type of a variable. That's great in this situation since we don't know what type `a` and `b` are since they were declared with `auto`. But the compiler will indeed assign them a type, and we've captured that type and will treat it as pseudo-variables `A` and `B`.

More importantly, we want the `decltype`s of `a[i]` and `b[j]`. These are the actual values stored in the `Array`s. In our current example, they could either be `float` or `double`.

Next we create a `constexpr bool`. This is not a boolean variable going on the stack, it's a constant expression evaluated at compile time. We assign them to `std::equality_comparable` because we obviously don't want to try comparing `a[i]` and `b[j]` as types that don't even have an `operator==`.

Now to our _constexpr if_ chain. First off, if our two types have an `operator==` between them, just call that and be done. If not, we can try converting one side to the other's type if that other type has an `operator==`. There's a question here though: in what order do we convert?

I made the executive decision that:
- If we can convert to one of the types with a nothrow conversion, do that before doing a conversion that throws.
- If there's conversions both ways, convert only the left hand side to the right hand side's type.

And of course, if there's no conversions and no `operator==`, just return `false`.

## External Storage

Now we're done talking about `VariantArray`. It works, and it does a surprising number of things at compile-time. On to `Datum`.

There's an extra level of complexity we're going to add to `Datum`: The ability to use external storage. What I mean is, we're going have the option to give `Datum` some memory that it doesn't own. Memory it can read and modify, but not deallocate or reallocate. Why? Let's look back at the markup where this all started, and at the same time peek ahead at what the markup will look like in our engine:

```c++
[[Attribute]]
int myInt;
```

The `[[Attribute]]` is a C++ tag that the compiler will ignore and our custom pre-parser (info coming in a future article) will look for. What will end up being constructed is a `Datum` who's storage will be that exact `int`. So the following pseudocode is equivalent:

```c++
myClass.myInt = 5;
myClass.GetDatum("myInt") = 5;
```

They both modify the same memory because the memory reference by the `Datum` _is_ the same memory being used to store `myInt` within the instance of the class.

So without getting too tangled up in the details of how all that is going to be wired up, let's take a step back and look at just what `Datum` needs to consider. We need to be able to give `Datum`:
- a pointer to an array
- a number of elements in that array
- a maximum number of elements in that array


All without have `Datum` ever trying to deallocate or reallocate that memory. Easier said then done. Keep in mind `Datum` is just a `VariantArray` which is composed of an `Array` which always assumes it owns its data. So we need to do two things:
- Keep track of when we're using internal/external storage.
- Be very careful when we're doing operations such as `PushBack` which could resize the `Array`.

We can accomplish the first one with a simple `bool`. Here's the data layout of `Datum`:

```c++
class Datum final : public VariantArray<bool, int, float, std::string, std::shared_ptr<RTTI>>
{
	bool isExternal{ false };
};
```

We've been very concerned with the sizes of our types so far, so why stop here. On an x64 platform, `sizeof(VariantArray)` is always 24 bytes. `Datum` will be that size, plus at least an extra byte for the `bool`, rounding out to 32 bytes due to padding. So that's an 8-byte bool. 64 bits for a value that really only needs one bit. You have no idea how much that irks me.

If we wanted to go _crazy_ we could do something called "bit stealing". Let's assume for a moment that `Datum` didn't store `bool`s. That would make `int` the smallest type `Datum` supports. Because `int`s are 4-bytes wide, they must always be stored at addresses that are multiples of 4, meaning all of those addresses will end in 00. That's two unused bits we can steal! `Array` has exactly one pointer in it, the pointer to the starting address. We could bitmask the last bit of that pointer to 1 whenever we're using external storage, and bitmask it back to 0 when we're not.

There's plenty of complications with this though. First off, it completely kills portability. Right now Windows x64 is the only platform I deem essential to support, but there's also nothing preventing this code from working on other platforms. I would like to keep this project open to as many platforms as possible. Furthermore, I _do_ want `Datum` to support `bool`, and if it does then those values will appear on 1-byte boundaries, meaning every bit of the pointer is used. I _could_ get around this by implementing `Array<bool>` as a bitarray like many stl implementations do, but I also don't want to limit `Datum` from supporting other 1-byte values like `char` or `std::byte`.

So I'm sticking with the 8-byte `bool` for now. 32-bytes is adequately small for such a powerful type as `Datum`. In the future I will be creating a memory manager/garbage collector for this engine and with that I might be able to create some other assurances about memory, such as perhaps that the most significant bit of an address will never be used and could be marked. At which point I would create a `TaggedPointer` type and make `Array` use that.

With that out of the way, let's get into the details of how this `bool` is used. For every method that can potentially resize the `Array`, we need to make a wrapper in `Datum` that first makes sure the container is not full when using external storage. I've encapsulated these into two methods:

```c++
void Datum::ThrowExternalIfFull() const
{
	if (IsFull())
	{
		ThrowExternal();
	}
}

inline void Datum::ThrowExternal() const
{
	if (IsExternal())
	{
		throw ExternalStorageException("Attempted to mutate memory of external Datum.");
	}
}
```

Cool. Now what about the methods we have to wrap? Aren't there a lot? Yes and no. Yes there's a lot of methods, but we actually don't have to write that many wrappers to encapsulate them. For example, here's `Insert`:

```c++
template<typename... Args>
inline auto Datum::Insert(Args&&... args)
{
	ThrowExternalIfFull();
	return VariantArray::Insert(std::forward<Args>(args)...);
}
```

This wraps all calls to `Insert`, whether they be templated or not, whether they have no arguments or twenty, whether they're void or return an iterator, they're all covered.

Most of the methods look like that. Very simple wrappers. What's more complicated are the special member functions. Luckily I was able to get away with using C++'s (rather obscure) ctor inheriting scheme:

```c++
// this inherits all ctors
// no option for only inheriting select ctors :(
// but I want all the ctors anyway, so that's not a big deal 
using VariantArray::VariantArray;
```

But I had to do some specific things for a few of them. Let's start with the dtor:
```c++
Datum::~Datum()
{
	if (IsExternal())
	{
		// zero everything so ~Array() won't do anything
		std::memset(this, 0, sizeof(Datum));
	}
}
```
This is a very dirty trick. After `~Datum()` is called, `~VariantArray()` will be called, then `~Array()`. `~VariantArray()` does nothing, but `~Array()` will attempt to free any memory it thinks it has. Keep in mind that `Array` has no concept of external storage, it thinks everything is internal. With no publicly exposed method for emptying without freeing, the only option is to forcibly zero everything out. This ends up harmlessly setting the pointer to `NULL`, and the `size`/`capacity` to 0.

Much more complicated is the copy-assignment operator. There's 4 cases to consider:

| this     | other    |
|:--------:|:--------:|
| internal | internal |
| internal | external |
| external | internal |
| external | external |

and I handle them like so:

```c++
Datum& Datum::operator=(const Datum& other)
{
	if (this != &other)
	{
		if (other.isExternal)
		{
			if (!isExternal)
			{
				Empty();
			}
			std::memcpy(this, &other, sizeof(Datum));
		}
		else
		{
			if (isExternal)
			{
				// zero everything so Array::operator= won't free anything
				std::memset(this, 0, sizeof(Datum));
			}
			VariantArray::operator=(static_cast<const VariantArray&>(other));
		}
	}
	return *this;
}
```

If `other` is external, we need to free any internal memory and then copy over the external memory without copying the contents of the memory itself.

If `other` is internal, we need to zero out any external memory like we did in the dtor before letting `VariantArray::operator=` handle it as if this were a default-constructed `Datum`.

## Conclusion

I'm extremely happy with how this container came out. Its better in every possible way compared to my original `Datum` which used tables indexed into at runtime rather than `std::variant` which accomplishes the same at compile-time. What's easily the most impressive parts are how `Datum` can support extra types by just editing a template parameter and that we can get compile-time implicit and automatic conversions to types not explicitly listed. This was a fun project that took full advantage of C++'s compile-time functionality. I would up writing so many templates that I started ending my sentences with angle brackets.

Thanks for reading>
