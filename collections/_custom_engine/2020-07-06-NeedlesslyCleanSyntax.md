---
title: Needlessly Complex Code to Achieve Needlessly Clean Syntax
author: Jarrett Wendt
excerpt: The story of my life in C++.
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
thumb: assets/images/CleanCode.webp
layout: post
cwd: '../'
---

{% assign vvv = site.custom_engine | where: 'title', 'Visiting Variant Vectors' | first %}
{% assign bindingToPython = site.custom_engine | where: 'title', 'Binding to Python' | first %}
{% assign establishingAHierarchy = site.custom_engine | where: 'title', 'Establishing a Hierarchy' | first %}

<figure>
<img src="{{ page.cwd }}{{ page.thumb }}" alt="WargamesTicTacToe" width="500" style="display: block; margin-left: auto; margin-right: auto;">
</figure>

Take a look at this Python code that is completely possible in my engine.
```python
from MyGameEngine import Entity, Time

class MyEntity(Entity.Entity):
	speed = 5

	def _Update(self):
		super()._Update()
		delta = speed * Time.Delta()
		self.localTransform.translation.x += delta
```

It's a very simple `Entity` that does nothing but move five units in the positive x direction every second. Let's more closely examine the last line:
```python
self.localTransform.translation.x += delta
```
This seemingly innocuous statement is the lid to an enormous can of worms. To understand the context, let's take a look back at <a href="{{ establishingAHierarchy.url }}" target="_blank">one of my previous posts</a> where I put a `Transform` on every <a href="{{ establishingAHierarchy.url }}" target="_blank">`Entity`</a> in my engine. This C++ code uses a get/set approach:

```c++
Transform Entity::GetLocalTransform();
Transform Entity::GetWorldTransform();
void Entity::SetLocalTransform(const Transform& t);
void Entity::SetWorldTransform(const Transform& t);
```

We like this get/set approach in C++ because it ensures we have the least number of Transform accumulations possible when a parent moves and all the children world transforms must be updated.

However, creating bindings that allows Python code looking like this:
```python
self.localTransform.translation.x += delta
```

When the same code in C++ looks like this:
```c++
Transform t = this->GetLocalTransform();
t.translation.x += delta;
this->SetLocalTransform(t);
```

Is exceedingly difficult. In the example Python above, we're actually calling two getters and a setter:
- `self.localTransform` getter
- `.translation` getter
- `.x +=` setter

So when should we invalidate this Entity's world transform? When we get `localTransform`? How do we know that we're getting it to do an assignment later? What if we were just getting it to read some date? It would horribly inefficient to always invalidate on reads. What we need to do is _defer_ invalidation until the last possible moment, which would be on any setter.

To accomplish this, these getters shan't return `Transform`s or `Vector3`s. Instead they need to return wrappers which further delay invalidation. These wrappers only need to refer back to the object they'll act on, while the actions in which they'll take are implicit through their type.

## Breaking down the problem in C++
If we're going to need to make a slew of wrappers in order to get the desired clean code in Python, we might as well make those wrappers accessible in C++ too so that we can get similar syntax such as:
```c++
myEntity->GetLocalTransform().Translation().X() += delta;
```

Let's start breaking this down...
```c++
// we return a wrapper, not an actual Transform
TransformWrapper Entity::Transform();

struct TransformWrapper final
{
	// only 8 bytes on x64 systems, not 64 bytes which is sizeof(Transform)
	Entity& owner;
	
	// implicit conversion to an actual Transform
	operator Transform() const;

	// properly invalidates the owner's cached world Transform
	TransformWrapper& operator=(const Transform& t);

	// get wrappers to sub-components...
	TranslationWrapper Translation();
	RotationWrapper Rotation();
	ScaleWrapper Scale();
}
```

This is a great starting point. The only data we need to store is a reference to the owner which is nice and small. An implicit conversion operator lets us treat this wrapper as if it's a real `Transform`. Then we have an assignment operator which successfully lets us delay invalidation by just one level.

However... then there's the sub-components. If we were to go along with this scheme, we'd need to have a separate class wrapping each of the translation, rotation, and scale.

What about one level deeper into the xyz components? Would we need a wrapper for them? Yes. In fact, we'd need a wrapper for each of them _for each Transform sub-type_. That means we'd need a `TranslationXWrapper`, `TranslationYWrapper`, `TranslationZWrapper`, `RotationXWrapper`, etc.

Plus we don't have just _one_ `Transform`, we have two: local and world. So just double all of those wrapper classes we have, because we'd need different behavior for the world and local versions.

And then if we want to be `const`-correct we'll need to duplicate all the classes again to make `const` and non-`const` versions similar to how the standard library has `iterator`s and `const_iterator`s for all their containers.

And _then_ there's a slew of operators we need to define for each class. All of the types which wrap a `Vector3` need to have every operator defined for `Vector3`. Similarly, all types which wrap a single `float` need to have an operator defined for every one that `float` has.

All-in-all, we're looking at _at least_ 52 distinct classes we need to define, each with at least a dozen operators to achieve complete functionality.

Yikes, that seems like a lot of typing. So how can we make this better? By using templates of course!

## Here begins the templated metaprogramming...
If you've been reading my blog, you'll know that I'm not afraid to use extremely whacky templates in order to achieve my goals. For <a href="{{ vvv.url }}" target="_blank">`Datum`</a> I had to pull out almost every trick in the book in order to create a custom variant container.

Let's start at the root-most level, where we first get our wrapper to a transform from an `Entity`:

```c++
enum class CoordinateSpace
{
	Local,
	World
};

template<CoordinateSpace Space>
TransformWrapper<Space, false> Transform();
template<CoordinateSpace Space>
TransformWrapper<Space, true> Transform() const;
```

What we're doing here is we're wrapping both the `CoordinateSpace` as well as `const`ness into the template parameters of the `TransformWrapper` class. You might be worrying that this will result in some long-winded method call on the C++ side:
```c++
myEntity->Transform<CoordinateSpace::Local>()
```
But in fact it can be shortened quite a bit with C++20's `using enum`:
```c++
using enum CoordinateSpace;
myEntity->Transform<Local>();
```
Which is much easer to read as "get the local transform from my entity".

The code for `TransformWrapper` itself isn't much different from what we saw before:
```c++
template<CoordinateSpace Space, bool IsConst>
struct TransformWrapper final
{
	// only 8 bytes on x64 systems, not 64 bytes which is sizeof(Transform)
	Entity& owner;
	
	// implicit conversion to an actual Transform
	operator Transform() const;
	
	// properly invalidates the owner's cached world Transform
	TransformWrapper& operator=(const Transform& t) requires (!IsConst);
	
	// get wrappers to sub-components...
	Vector3Wrapper<Space, Transform::Component::Translation, IsConst> Translation();
	Vector3Wrapper<Space, Transform::Component::Scale, IsConst> Scale();
	QuaternionWrapper<Space, IsConst> Rotation();
};
```

You can see here that by using C++20 constraints we can delete `operator=` when this is a "const" `TransformWrapper`.

When getting the sub-components, we pass along the templated `CoordinateSpace` and `const`ness. Since `Scale` and `Translation` are both `Vector3`s I decided to have them share the same wrapper type to reduce duplicate code. Unfortunately, since `Rotation` is stored as a `Quaternion` it must have its own wrapper. It's possible I could have created a single `Vector4Wrapper` that all of these share but I'm predicting that one day these different types will have such different behavior they'll need to be separate classes anyway.

Looking a level deeper, this is what `Vector3Wrapper` looks like:
```c++
template<CoordinateSpace Space, Transform::Component Type, bool IsConst>
struct Vector3Wrapper final
{
	// still only 8 bytes :)
	Entity& owner;

	// implicit conversion to an actual Vector3
	operator Vector3() const;

	// properly invalidates the owner's cached world Transform
	Vector3Wrapper& operator=(const Vector3& v) requires (!IsConst);

	// get the sub-components by name
	FloatWrapper<Space, Type, 0, IsConst> X();
	FloatWrapper<Space, Type, 1, IsConst> Y();
	FloatWrapper<Space, Type, 2, IsConst> Z();

	// all the operators Vector3 has...
	Vector3Wrapper& operator+=(float f) requires (!IsConst);
	Vector3Wrapper& operator+=(const Vector3& v) requires (!IsConst);
	Vector3Wrapper& operator-=(float f) requires (!IsConst);
	// ...
};
```

This looks extremely similar to what we did with `TransformWrapper`. We of course have our nifty implicit conversion operator and assignment operator. There's also a definition for every operator that `Vector3` has (which is unfortunately a lot). For any compound-assignments such as `operator+=` they will of course invalidate the owner's cached world `Transform` just like `operator=`.

Also similarly to `TransformWrapper` we once again pass along our template arguments to the sub-component wrapper types and then contextually providing new values per-method.

Let's take a look at the final level of indirection. This should look very familiar by now...
```c++
template<CoordinateSpace Space, Transform::Component Type, size_t Index, bool IsConst>
struct FloatWrapper final
{
	// still only 8 bytes :)
	Entity& owner;

	// implicit conversion
	operator float() const;

	// properly invalidates the owner's cached world Transform
	FloatWrapper& operator=(float f) requires !(IsConst);

	// all of the operators float has (which is a lot)...
	FloatWrapper& operator+=(float f) requires (!IsConst);
	FloatWrapper& operator-=(float f) requires (!IsConst);
	FloatWrapper& operator*=(float f) requires (!IsConst);
	FloatWrapper& operator/=(float f) requires (!IsConst);
	// ...
}
```

Now that we're finally at the bottom level we can see how all the information needed for knowing exactly _which_ `float` in our `Transform` needs to be modified can be known at compile-time. We know whether it's local or world by using `CoordinateSpace`, we know if it's translation, rotation, or scale with a `Transform::Component`, we know which component to get out of that vector via an `Index` and we know whether or not to allow certain operators with our boolean `const`ness.

I always get excited when I can find ways to let the compiler do all of the heavy lifting. This also keeps our data types small since the only thing all of these wrappers need is a reference to the `Entity` they came from. Assuming the compiler implements that as a pointer, that would be only 8 bytes on a 64-bit system.

## A tangent with compile-time integrals
It's all well and good that we can have this line of code now in C++:
```c++
myEntity->Transform<Local>().Translation().X() += 5;
```
But what if the algorithm we're working with is easier to think of in terms of indices?
```c++
myEntity->Transform<Local>().Translation()[0] += 5;
```
Can we have an `operator[]` that let's us do that? Let's try...
```c++
auto Vector3Wrapper::operator[](size_t index)
{
	return FloatWrapper<Space, Type, index, IsConst>(owner);
}
```
Unfortunately, this code will not compile because `index` is a variable. Its value is not known by the compiler. The compiler _needs_ to know the value of it before it can instantiate a `FloatWrapper`. You've likely encountered similar problems why trying to make a dynamically-sized `std::array`. This code will not compile for the same reasons:
```c++
size_t size = 5;
std::array<int, size> myArray;
```

So how can we get around this? Surely there _must_ be some way to get this to work at compile-time. I mean, we're typing the index directly into the source code. It wasn't originally a variable, it was a literal, which is something the compiler _can_ deal with. So how can we keep it as a literal and not let it get stuffed into a variable?

Enter <a href="https://en.cppreference.com/w/cpp/types/integral_constant" target="_blank">`std::integral_constant`</a>. This is a type which can encapsulate a compile-time known value. Implementing our `operator[]` with it looks like this:
```c++
template<size_t Index>
FloatWrapper Vector3Wrapper::operator[](std::integral_constant<size_t, Index>)
{
	return FloatWrapper<Space, Type, Index, IsConst>(owner);
}
```
We don't actually care about the `std::integral_constant` being passed in; we only care about one of its template parameters: `Index`. Now that we have that index at compile-time, we can use it as the final piece to instantiate a `FloatWrapper`.

However, calling this method is a little long-winded...
```c++
myEntity->Transform<Local>().Translation()[std::integral_constant<size_t, 0>()] += 5;
```
Honestly, if I have to manually construct a `std::integral_constant` every time, I'm just going to be using `.X()` instead. `std::integral_constant` is a type that exists on the more weird and obscure side of C++ and I'd rather not forcibly expose my users to it. Ideally, we could just define a `std::integral_constant` literal like so:
```c++
myEntity->Transform<Local>().Translation()[0_zc] += 5;
```
Where `z` is for si<u>Z</u>e_t and `c` is for <u>C</u>onstant. Great! Literal operators are really easy to define in C++, so let's make on really quick.

```c++
constexpr auto operator""_zc(unsigned long long int x)
{
	return std::integral_constant<size_t, x>();
}
```
But there's a problem here. The same problem yet again. We're using a variable as a template parameter, which is illegal in C++. Guess our journey ends here then since all C++ literal operators are pass-by-value...

<img src="{{ page.cwd }}assets/images/ThereIsAnother.jpg" alt="ThereIsAnother" width="400" style="display: block; margin-left: auto; margin-right: auto;">

But there is A New Hope! C++ has a another lesser-known type of literal operator. A _templated_ one:
```c++
template<char... Digits>
constexpr auto operator""_zc();
```

What this function takes is the array of characters as a variadic template argument. To help clarify, these two function calls would be identical:
```c++
12345_zc
operator""_zc<'1', '2', '3', '4', '5'>();
```
So to make this work we need to parse the variadic template arguments into an integer. This is mostly the same algorithm as `atoi` which is an exercise most of us have already implemented in undergrad. The trick now is that it needs to be done _at compile time_.

I'll defer the rest of this tangent to <a href="https://gist.github.com/mattbierner/5c698972de0cdd9de86a" target="_blank">Matt Bierner</a> who has some excellent code that gets this done. It can even handle binary, octal, and hexadecimal literals!

## Binding to Python
So now that we have all of our wrappers implemented on the C++ side, now we need to write some bindings so that we can get equally clean syntax in Python.

I'm only going to show `Vector3Wrapper` since it's mostly the same as `TransformWrapper` and `QuaternionWrapper`. Luckily, we don't need any `FloatWrapper`s since Python has its own getters and setters similar to how properties work in C#. If you're used to using straight C99 syntax when using the Python C API, <a href="{{ bindingToPython.url }}" target="_blank">check out my article on how I'm using it with C++ syntax</a>.
```c++
namespace py
{
	struct Vector3Wrapper : public PyObject
	{
		// not necessarily local, we force cast to the value of space
		// not necessarily scale, we force cast to the value of component
		Entity::Vector3Wrapper<CoordinateSpace::Local, Transform::Component::Scale> v;
		CoordinateSpace space;
		Transform::Component component;

		PyFloatObject* GetX();
		PyFloatObject* GetY();
		PyFloatObject* GetZ();

		int SetX(PyFloatObject* value);
		int SetY(PyFloatObject* value);
		int SetZ(PyFloatObject* value);

		static inline PyGetSetDef getset[]
		{
			{
				"x",
				Util::UnionCast<getter>(&Vector3Wrapper::GetX),
				Util::UnionCast<setter>(&Vector3Wrapper::SetX),
				"x component"
			},
			{
				"y",
				Util::UnionCast<getter>(&Vector3Wrapper::GetY),
				Util::UnionCast<setter>(&Vector3Wrapper::SetY),
				"y component"
			},
			{
				"z",
				Util::UnionCast<getter>(&Vector3Wrapper::GetZ),
				Util::UnionCast<setter>(&Vector3Wrapper::SetZ),
				"z component"
			},		
			{ nullptr }
		};

		static PyTypeObject type
		{
			.ob_base = { PyObject_HEAD_INIT(nullptr) 0 },
			.tp_name = "Vector3Wrapper",
			.tp_basicsize = sizeof(Vector3Wrapper),
			.tp_itemsize = 0,
			.tp_flags = Py_TPFLAGS_DEFAULT,
			.tp_doc = "Python port of C++ Entity::Vector3Wrapper",		
			.tp_getset = getset,
			.tp_new = PyType_GenericNew,
		};
	};
}
```

You'll notice that there's only 3 data members:
1. Our original `Vector3Wrapper` which we made before.
2. The `CoordinateSpace`.
3. The `Transform::Component`.

But why are we storing the `CoordinateSpace` and `Transform::Component`? Don't those exist in the template information of `Vector3Wrapper`? Isn't that why we just went through the whole rigmarole before?

<img src="{{ page.cwd }}assets/images/WellYesButActuallyNo.jpg" alt="ThereIsAnother" width="400" style="display: block; margin-left: auto; margin-right: auto;">

Unfortunately, Python doesn't respect our templates. This isn't a flaw in Python, it's just that Python and C++ have completely different type systems. C++'s is (almost) entirely compile time while Python's is (almost) entirely runtime. Creating a templated `PyObject` would be pointless because that template information would not persist within the Python interpreter's runtime. It's still possible for us to make this work _entirely_ compile-time with no additional data members or branches, but it would require us to go back to exhaustively implementing all 50+ permutations of these templated classes as individual non-templated classes.

Here's an example of how just one of the methods on this binding look:
```c++
PyFloatObject* py::Vector3Wrapper::GetX()
{
	using enum CoordinateSpace;
	using enum Transform::Component;

	float f;
	if (space == Local)
	{
		if (component == Scale)
		{
			f = reinterpret_cast<Vector3Wrapper<Local, Scale>&>(v).X()
		}
		else
		{
			f = reinterpret_cast<Vector3Wrapper<Local, Translation>&>(v).X()
		}
	}
	else
	{
		if (component == Scale)
		{
			f = reinterpret_cast<Vector3Wrapper<World, Scale>&>(v).X()
		}
		else
		{
			f = reinterpret_cast<Vector3Wrapper<World, Translation>&>(v).X()
		}
	}
	return PyFloat_FromDouble(f);
}
```
In the full implementation I defer the nested `if`-`else` statements to a separate helper method but you get the point.

What we've done is create some beautiful templated wrappers, only to hack it apart into separate variables, and then hot glue it back together to get our original template back just so we can invoke the proper method. Our fingers are burnt and the Python side isn't very pretty, but the final product is functional and has a very minimal amount of duplicate code.
