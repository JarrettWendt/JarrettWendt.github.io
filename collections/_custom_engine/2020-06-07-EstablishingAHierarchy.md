---
title: Establishing a Hierarchy
author: Jarrett Wendt
excerpt: Finally creating something familiar...
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
layout: post
cwd: '../'
---
{% assign reflection = site.custom_engine | where: 'title', 'Reflection in C++' | first %}

Let's look at how some of the major game engines organize their scenes/levels in object-oriented fashions.

In Unreal, we have Actors that may have several Components attached. There exists Child Actor Components, meaning we can have a recursive hierarchy.

In Unity, we have GameObjects that can naturally have many children GameObjects and 1 parent GameObject. A GameObject can have as many MonoBehaviours attached as they want.

In Godot, everything is a Node. A Node has 1 parent and as many children as you want.

Personally, I like the flexibility Godot's system provides. You never have to worry about if the feature you're implementing should be an entity or component, and you'll never have to worry about migrating code to be one or the other later if your initial decision was wrong.

Thus, I've decided to model my hierarchical system to be similar in philosophy to Godot's. In my system, everything is an `Entity`. An `Entity` has one or no parents and many children.

In a previous implementation of my engine, this hierarchy was kept through raw pointers stored in a `HashMap`. As the engine got more complex and more features were built on top of this hierarchy, I became increasingly paranoid about the potential for dangling pointers, particularly introduced whenever that `HashMap` is resized without warning. So in this new implementation of the engine, I've decided to instead use smart pointers. In fact, an `Entity` can only _exist_ as a `std::shared_ptr`. It is undefined behavior to instantiate an `Entity` with anything other than `std::make_shared`. We'll see why in a minute.

The basic requirements I wanted for my `Entity` are as follows:
- Inherits from <a href="{{ reflection.url }}" target="_blank">`Attributed`</a>.
- To be able to iterate through children easily and efficiently.
- To be able to add/remove children from their parents.
- To be able to look up children by name.
- To be able to re-parent children to a different `Entity`.
- To have `Update`/`Init` methods on `Entity` which recursively invokes `Update`/`Init` on all children.
- To be able to enable/disable Entities such that their `Update`/`Init` is not called.
- For `Entity` to have a efficient hierarchical `Transform` that establishes their position in the scene.

## Child Services
The principal methods for dealing with children are the aptly named `Adopt()` and `Orphan()`. Keeping in mind that each `Entity` has a `HashMap<std::string, std::shared_ptr<Entity>>`, a simplified version of `Adopt()` looks like this:

```c++
std::shared_ptr<Entity> Entity::Adopt(const std::string& childName, SharedEntity child)
{
	const auto [it, inserted] = children.Insert(childName, child);
	if (!inserted) [[unlikely]]
	{
		throw InvalidNameException("child with name " + childName + " already exists");
	}
	child->name = childName;
	child->parent = weak_from_this();
	return child;
}
```

So the first weird thing that might jump out at you is the `[[unlikely]]` attribute. This is an attribute that hints to the compiler that you don't expect this `if` statement to be true very often. This helps the compiler generate better branch predictions.

The next strange thing is `weak_from_this()`. The method does exactly what it sounds like. It returns a `std::shared_ptr` to `this`. So why couldn't we have just done `std::shared_ptr<Entity>(this)`? Because doing so would create a completely _separate_ `std::shared_ptr` with a completely separate reference count from any existing `std::shared_ptr` to `this`. So we need to get a `std::shared_ptr` from some existing copy of the `std::shared_ptr` that already references `this`. This might sound impossible to do from within ths scope of a method where all we have is a raw pointer `this` but luckily the standard library has our back!

There's a base class called <a href="https://en.cppreference.com/w/cpp/memory/enable_shared_from_this" target="_blank">`std::enable_shared_from_this`</a>. It provides two public methods: `shared_from_this()` and `weak_from_this()` It's (likely, based on your compiler) implemented by having a single member variable that is a `std::weak_ptr` to the original `std::shared_ptr` that this object was instantiated with. That `std::weak_ptr` is undefined if the object was instantiated with anything except `std::make_shared`. So that means an `Entity` can never be instantiated with `new` or even be instantiated on the stack. I believe this is a small syntactical price to pay since for the most part Entities will be instantiated using factory helper methods such as `CreateChild`.

Since I want every child to have a reference back to their parent, it makes sense for that reference to be a `std::weak_ptr`. It shouldn't be a `std::shared_ptr` because then I would have a cyclic dependency where neither object's reference count ever reaches 0, so neither is ever destroyed.

With the complexities of smart pointers in `Adopt()` out of the way, `Orphan()` is _child's play_. We simply remove the entry from the `HashMap` and remove the reference to the parent.
```c++
std::shared_ptr<Entity> Entity::Parent() noexcept
{
	// lock() returns nullptr if the weak_ptr has gone bad
	// i.e. the shared_ptr hit a refcount of 0
	return parent.lock();
}

// Orphan's this child from its parent.
void Entity::Orphan() noexcept
{
	if (auto p = Parent())
	{
		p->children.Remove(name);
		parent = {};
	}
}
```

## Hierarchical Transforms
When using smart pointers, establishing an object tree like this turned out to be rather simple. What was a little more complex was developing an efficient use of `Transform`s in this hierarchy. A `Transform` in game programming is an object that represents a translation, rotation, and scale. They're often implemented with either a single transformation matrix, as three `Vector3`s, or as two `Vector3`s and a `Quaternion`. The exact implementation of a `Transform` is not important at this juncture, but what is important is recognizing that changing the `Transform` of a parent can potentially impact all child `Transform`s and that could be a slow operation.

You can think of any object as having two `Transform`s: a `localTransform` and a `worldTransform`. The `localTransform` describes an object's transformation in relation to it's parent. The `worldTransform` describes an object's transformation in relation to the coordinate system's origin.

On my `Entity` I have four methods in regard to `Transform`s:
- `GetLocalTransform()`
- `GetWorldTransform()`
- `SetLocalTransform()`
- `SetWorldTransform()`

As the most naïve approach, we could store a `localTransform` on every `Entity` and then whenever we call `GetWorldTransform()`, we calculate it on the fly by accumulating all `localTransform`s of all parents. This method should sound woefully inefficient to anyone reading this. If we're simply reading and not modifying the `Transform`s, recalculating the `worldTransform` on demand every time is a huge waste of time.

The first step to optimization here is some simple memoization. We store not only a `localTransform` on every `Entity`, but also their `worldTransform`. I expect a `Transform` to be no bigger than 64 bytes on an x64 machine, so the memory cost here should be worth it. The problem now is what to do when we manipulate a `Transform`. If we change either the local or world `Transform`, all memoized `worldTransform`s in the children need to be recalculated. This could result in atrocious performance if the user is changing `Transform`s of the parents often but rarely looking at the children.

The best thing to do here is to defer recalculation until the last possible moment, only calculating it when we need it. We can do this by simply storing a `bool` that indicates whether or not the memoized `worldTransform` is valid.

So on `SetLocalTransform()` and `SetWorldTransform()` we recursively go through all children and set their `bool`s to false.

On `GetWorldTransform()` we return the cached `worldTransform` only if we know it's valid, otherwise we must recalculate it.

This minimizes the number of redundant `Transform` accumulations to the smallest amount possible. The most concerning expense now is the recursive descent into all children whenever a `Transform` is manipulated. This could be a costly operation if it results in a lot of cache misses. I'm not too worried about this though for two reasons:
1. We would have had those cache misses anyway if we were using the naïve approach.
2. Those cache misses could be mitigated using a custom memory manager that colocates all Entities in memory, and optimization I intend to implement.

I've delved into the source code for Unreal engine for inspiration on my implementation and have found that they take a similar approach. Whenever you call a method such as `SetActorLocation()`, a recursive descent is made into all child components to mark them for an update. I've also looked into some disassembly for Unity and found they do this as well, though it's much more abstracted thanks to C#'s Properties.
