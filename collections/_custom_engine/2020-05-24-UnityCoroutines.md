---
title: Reimplementing Unity Coroutines in C++
author: Jarrett Wendt
excerpt: Insert blerb about how all programs need to be multithreaded now, the slowdown of Moore's Law, all the cores on Ryzen CPUs, etc.
sourceRepo: https://github.com/JarrettWendt/FIEAEngine
layout: post
cwd: '../'
---

> Insert blerb about how all programs need to be multithreaded now, the slowdown of Moore's Law, all the cores on Ryzen CPUs, etc.

Unity provides a really convenient way to add asynchronous logic into your code:
```c#
using UnityEngine;
using System.Collections;

public class ExampleClass : MonoBehaviour
{
	private void Start()
	{
		StartCoroutine(RunsOnceASecond);
	}

	private IEnumerator RunsOnceASecond()
	{
		yield return new WaitForSeconds(1);
		Debug.Log("RunsOnceASecond " + Time.time);
	}
}
```

Personally, I prefer to define my coroutines as a local function if they're small and only used once:
```c#
using UnityEngine;
using System.Collections;

public class ExampleClass : MonoBehaviour
{
	private void Start()
	{
		StartCoroutine(runsOnceASecond);
		IEnumerator runsOnceASecond()
		{
			yield return new WaitForSeconds(1);
			Debug.Log("RunsOnceASecond " + Time.time);
		}
	}
}
```
Unfortunately, you can't pass a lambda into `StartCoroutine()`.

To be clear, this isn't _truly_ asynchronous. All of Unity's coroutines run synchronously, just like `Start()` and `Update()` methods. So coroutines won't help you improve performance through threading, but they are useful for breaking up your code to be _logically_ asynchronous. Good for when you need something to run $n$ seconds from now, or every $n$ seconds.

The alternative in Unreal is to use their Timers. Its much less expressive but provides much more control:
```c++
void AMyActor::BeginPlay()
{
	Super::BeginPlay();
	// Call RepeatingFunction once per second, starting two seconds from now.
	GetWorldTimerManager().SetTimer(MemberTimerHandle, this, &AMyActor::RepeatingFunction, 1.0f, true, 2.0f);
}

void AMyActor::RepeatingFunction()
{
	// Once we've called this function enough times, clear the Timer.
	if (--RepeatingCallsRemaining <= 0)
	{
		GetWorldTimerManager().ClearTimer(MemberTimerHandle);
		// MemberTimerHandle can now be reused for any other Timer.
	}
}
```

It's also important to know that, in Unity's Coroutines and Unreal's Timers are attached to some sort of object. So when that object dies, the asynchronous code does too. This turns out to work great for most use cases, such as if you were to have a heart rate monitor on your main character which needs to be updated at some steady interval. You wouldn't want that heart rate code to continue in a scene where the main character object doesn't exist. However, there are ways to get your asynchronous code to exist in a more detached state. In Unreal, you can bind a timer to a static delegate or a lambda. In Unity, you would have to create some other GameObject which only exists to hold the coroutine.

## My Interface

For my Engine, I decided to create a global manager for coroutines. So they're never implicitly attached to any object. If you want to make sure the coroutine dies when the object does, you must explicitly stop it in the destructor. This has been working so far, but if I decide to make it more Unity-like it shouldn't be difficult to adapt the code.

I had three main goals with my coroutine system:
- I wanted starting a coroutine to be simple. Just one call to a `Start()` function and that's it. No nonsense with handles or delegates like in Unreal.
- I didn't want to be limited in what kinds of functions I can pass in. Lambdas, free functions, and methods should all be fair game.
- I wanted simple syntax for yielding the coroutine similar to the combination of the C# feature `yield return` and the Unity object `WaitForSeconds`.

The last goal would be the most tricky because, for the most part, it's out of my hands and in those of the language designers of C++. Luckily, this exact syntax exists in C++20 with the new keywords `co_return` and `co_yield`. The end syntax ends up looking like:

```c++
Coroutine myCoroutine()
{
	// do something
	// ...
	// wait for half a second
	co_yield Time::Seconds(0.5);
	// do something else
	// ...
	// done
	co_return;
}

Coroutines::Start(myCoroutine);

```

So a coroutine function must return a `Coroutine` object, we yield a `Time::Seconds` object, and `Coroutines::Start()` takes any sort of a function that returns a `Coroutine`. This can also be used with lambdas:

```c++
Coroutines::Start([]()->Coroutine
{
	// do something
	// ...
	// wait for half a second
	co_yield 0.5s;
	// do something else
	// ...
	// done
	co_return;
});
```

Notice that we're explicitly declaring the lambda's return type with `->` (parenthesis are optional in a lambda that takes no parameters, unless you want to explicitly declare the return type). Also notice that we're not explicitly returning a `Time::Seconds`. Instead, we're taking advantage of std::chrono_literals for even cleaner syntax. We could just as easily write `co_yield 500ms` to achieve the same effect.

The above calls to `Start()` are a sort of "set-and-forget" version. But if you want to explicitly stop a coroutine later, you can do this:
```c++
Coroutines::Start("myCoroutine", []()->Coroutine
{
	// On no! This will go on forever!
	for(;;)
	{
		co_yield 1ms;
	}
	co_return;
});

// Not if I have anything to say about it...
Coroutines::Stop("myCoroutine");
```

## My Implementation

So how does this work? This is some very strange syntax since we're never actually returning a `Coroutine` object. So how is one made? Is one ever actually made?

When the function is first called, a `Coroutine::promise_type` object is constructed. `co_yield` and `co_return` subsequently invoke methods on that `promise_type`. The above coroutine, somewhat decompiled, looks like this pseudocode:

```c++
Coroutine::promise_type promise;
pseudo_return promise.get_return_object();
// execution continues beyond the pseudo_return
promise.initial_suspend();
try
{
	// do something
	// ...
	// wait for half a second
	promise.yield_value(0.5s);
	// do something else
	// ...
	// done
	promise.return_void();
}
catch (...)
{
	promise.unhandled_exception();
}
// C++ doesn't have a finally block, but pretend it does for a moment
finally
{
	promise.final_suspend();
}
```

So your return type, whatever that may be, must have a `::promise_type` sub-type. On that `promise_type`, you must define at least 6 methods:
- `get_return_object()`: this lets you get a handle to the coroutine so that you may resume it at a later time.
- `initial_suspend()`: can return either `suspend_always` or `suspend_never`. This defines the behavior of your coroutine when it is first called. Does it pause immediately or does it continue until the first `co_` statement?
- `yield_value()` or `yield_void()`: this is called whenever you `co_yield`.
- `return_value()` or `return_void()`: this is called whenever you `co_return`.
- `unhandled_exception()`: this is called whenever an exception is thrown. You can retrieve the exception by calling `std::current_exception()`. If you want to deal with exceptions at all in your coroutine, they _must_ be handled here. The exception will not be percolated up the call stack to whatever's resuming the coroutine.
- `final_suspend()`: can return either `suspend_always` or `suspend_never`. This is called whenever your function ends, whether that be from an exception or `co_return`. Like `initial_suspend`, it can further define the behavior of our coroutine. Such as if we want to start the function all over again or be done with it.

Let's start from the beginning. My `get_return_object()` looks like this:
```c++
using Handle = std::experimental::coroutine_handle<promise_type>;
auto Coroutine::promise_type::get_return_object() noexcept
{
	return Handle::from_promise(*this);
}
```

Okay, so my `get_return_object()` returns a `coroutine_handle` to the `promise_type`. I also made my `Coroutine` implicitly constructable from one of these handles:

```c++
Coroutine(promise_type::Handle handle) noexcept :
	handle(handle)
{
	assert(handle);
}
```

Hopefully you can see where this is going. So in `Coroutines::Start` I invoke the function, get this weird C++ thing called a `coroutine_handle`, convert it into my nice `Coroutine` type, and hold on to references of those. Now we can start taking a peek into `Coroutines`...

While `Coroutine` (singular) is the return type of a function that _is_ a coroutine, `Coroutines` (plural) is a static class that exists as a container of all `Coroutine` instances. The basic data layout of `Coroutines` looks like this:
```c++
class Coroutines final
{
public:
	STATIC_CLASS(Coroutines)

	using Key = std::string;
	using Functor = std::function<Coroutine()>;

private:
	using Pair = std::pair<std::shared_ptr<Functor>, std::shared_ptr<Coroutine>>;
	static inline HashMap<Key, Pair> coroutines{};
}
```

And the all-important `Start()` method:
```c++
void Coroutines::Start(const Functor& coroutine)
{
	// Counter from which pseudo-unique IDs are generated when none is provided.
	// If this doesn't work out, we could switch to GUIDs.
	static size_t uniqueCounter{};
	Start(std::to_string(uniqueCounter++), coroutine);
}

void Coroutines::Start(const Key& key, const Functor& coroutine)
{
	const auto func = std::make_shared<Functor>(coroutine);
	const auto coro = std::make_shared<Coroutine>(func->operator()());
	coroutines.Insert(key, { func, coro });
}
```

The weirdest thing here that might stand out to you is my usage of smart pointers. Let me explain, by first posing a problem:

```c++
void StartMyCoroutinePlease(Object* o, float seconds)
{
	Coroutines::Start([o, seconds]()->Coroutine
	{
		co_yield Time::Seconds(seconds);
		o->DoSomething();
		co_return;
	})
}
```

What will happen when we leave the scope of `StartMyCoroutinePlease()`? The local stack variables `o` and `seconds` will be destroyed. But that's okay because we passed them by value to the lambda... right?

First let's wonder _what is a lambda_? A lambda is nothing more than an anonymous struct. This struct has an `operator()` defined by the lambda's signature, in our case taking no arguments and returning a `Coroutine`. But this struct _also_ has _data members_. These members are defined by the capture list. So this lambda, pseudo-decompiled into it's true struct form, would look like this:

```c++
// They're usually given some garbage unique name by the compiler.
// You've probably seen something like this once or twice in an error message.
struct anonymous_lambda_12342908357120349
{
	Object* o;
	float seconds;

	Coroutine operator()()
	{
		co_yield Time::Seconds(seconds);
		o->DoSomething();
		co_return;
	}
}
```

Okay so `o` and `seconds` are member variables. Cool. What difference does that make?

It makes a _world_ of difference.

Those member variables _live with the object_. So for as long as the lambda might be called, we need to keep the lambda alive. In `Coroutines::Start()` I'm capturing the lambda into a `std::function` We _cannot_ just use that `std::function` to get our `Coroutine` handle and let it fall off the stack. That will destroy the member variables defined by the capture list.

Okay, so we have to hold on to a `std::function` as well as a `Coroutine` in the `HashMap`. But not just _a_ `std::function`, _the_ `std::function`. Not a copy, not a move, _the exact same one_ that was used to get the `Coroutine` in the first place. This has to do with how methods are implemented in C++. Every method, even void ones like our `operator()`, secretly take at least one parameter: the `this` pointer. It is through this implicit `this` pointer that our `operator()` knows what `o` and `seconds` we're talking about. So if we were to get our `Coroutine` from the lambda and then store a _copy_ of the lambda with it, then when we resume that `Coroutine`, it will still be looking for the original `this` pointer.

That's why I first store the `Functor` into a `std::shared_ptr`. Then I use that `Functor` in the `std::shared_ptr` to retrieve my `Coroutine`. This ensures that the original `Functor` that produced the `Coroutine` stays alive.

Now that we've looked at the beginning of a `Coroutine`'s life, and we've ensured that it has everything it needs to stay alive, let's skip ahead to the end of its life:

```c++
void Coroutines::Stop(const Key& key)
{
	coroutines.Remove(key);
}
```
Pretty simple.

With that out of the way, how do we _resume_ a `Coroutine`? For this, let's take a look at `Coroutines::Update()`. This is a private method that will be invoked only by the main engine loop.

```c++
void Coroutines::Update()
{
	for (auto& [key, pair] : coroutines)
	{
		if (!coro->Resume())
		{
			Stop(key);
		}
	}
}

/**
* @returns	true if this Coroutine has more work to do 
*/
bool Coroutine::Resume()
{
	if (!handle.done() && nextResume <= Time::CurrentTime())
	{
		handle.resume();
		nextResume = Time::CurrentTime() + handle.promise().yieldValue;
	}
	return !handle.done();
}
```

So we loop through all the `Coroutine`s and invoke their `Resume()` method. If their time is up, we resume them through the `Handle`. Afterwards, we figure out the next time this `Coroutine` should be resumed by looking at the `promise_type`'s `yieldValue` member, which is set in `promise_type::yield_value()`. Finally, if the `Handle` reports that it is `done()` which is only true after `final_suspend()` returns `suspend_always`, we remove the `Coroutine` from the `HashMap`.

## Issues
Can you spot the problem in `Coroutines::Update()`? What would happen when we remove something from a `HashMap` while we're iterating through it? Bad things, most likely. Since we're using a range-based for loop, our iteration is tied to `HashMap::iterator`s. While we could just look at how I implemented `HashMap` and read my documentation to figuring what (if anything) could go wrong, it's better for us to assume (and this holds true in general for any C++ container) that as soon as you modify the container, all iterators are invalidated. There are exceptions to this of course, many of them for the stl containers even listed on [cppreference](https://en.cppreference.com/w/), but now isn't a time to rely on exceptions to the rule because we have a bigger problem.

What happens if we call `Coroutines::Stop()` from within a coroutine? What happens if we call `Coroutines::Start()`? Yikes, what if we call `Coroutines::StopAll()` which `Clear()`s the `HashMap`?

Ladies and gentleman, we have a race condition. Well, sort of. It's race-condition-esque. It brings up feelings of race conditions. But it's not a real race condition because we're dealing with synchronous code. Luckily, that means we won't need locks to solve this problem (in fact, if we tried to use them we'd deadlock immediately). Race-condition-esque problems like this come up often with coroutines, since we _are_ dealing with asynchronous code (we can't be sure _exactly_ when it's going to be executed), but they're executed entirely synchronously.

The solution to this is to defer modifications of the HashMap to one point in time. We will do this with a _pending list_.
```c++
struct PendingOp final
{
	enum class Type : uint8_t
	{
		Add,
		Remove,
		RemoveAll
	};

	Pair pair;
	Key key;
	Type type;
};
static inline SList<PendingOp> pendingOps{};
```
A `PendingOp` contains all the information we need to add or remove something or everything from the `HashMap`. No method will directly modify the `HashMap`. Instead, they'll queue their operation into the pending list, which will be dealt with inside `Coroutines::Update()` before invoking any of the `Coroutine`s. An important bonus of using a singly-linked-list is that the order of operations is preserved. We would expect the following order of operations to result in a net add of 0 `Coroutine`s:
```c++
Coroutines::Start("_", []()->Coroutine { co_return; });
Coroutines::Stop("_");
```
Similarly, this would net 0 additions too:
```c++
Coroutines::Start("_", []()->Coroutine { co_return; });
Coroutines::Start("__", []()->Coroutine { co_return; });
Coroutines::StopAll();
```

For neatness, we apply all of these pending operations in `ApplyPending()` which is called on the first line of `Update()`:
```c++
void Coroutines::ApplyPending()
{
	for (auto& [pair, key, type] : pendingOps)
	{
		switch (type)
		{
		case PendingOp::Type::Add:
			coroutines.Insert(key, pair);
			break;
			
		case PendingOp::Type::Remove:
			coroutines.Remove(key);
			break;
			
		case PendingOp::Type::RemoveAll:
			coroutines.Clear();
			coroutines.Resize(1);
			break;
			
		default:;
		}
	}
	pendingOps.Clear();
}
```

# Actual Threading
This coroutine system so far is nothing but a convenient way to delay/repeat code. Let's step it up a notch and introduce a way to _actually_ run a coroutine asynchronously if we so choose. Ideally, the external interface should keep the same syntax, but now we can just pass a `bool` that, when `true`, will execute the `Coroutine` asynchronously.
```c++
Coroutines::Start(MyAsynchronousCoroutine, true);
```

To do this, we're going to need to keep separate the _async_ coroutines from the non-asynchronous or _blocking_ coroutines:
```c++
static inline HashMap<Key, Pair> blockCoroutines{};
static inline HashMap<Key, Pair> asyncCoroutines{};
```

We'll also need to protect access to the _pending list_ because the user might call `Start()`, `Stop()`, or `RemoveAll()` from within an async coroutine. We'll do this with a `std::mutex` and `std::scoped_lock`

```c++
static inline std::mutex mutex{};

void Coroutines::Start(const Key& key, const Functor& coroutine, const bool async)
{
	std::scoped_lock lock(mutex);
	const auto func = std::make_shared<Functor>(coroutine);
	const auto coro = std::make_shared<Coroutine>(func->operator()());
	pendingOps.PushBack({ { func, coro }, key, PendingOp::Type::Add, async });
}

void Coroutines::Stop(const Key& key)
{
	std::scoped_lock lock(mutex);
	pendingOps.PushBack({ { nullptr, nullptr }, key, PendingOp::Type::Remove });
}

void Coroutines::StopAll()
{
	std::scoped_lock lock(mutex);
	pendingOps.Clear();
	pendingOps.PushBack({ { nullptr, nullptr } , "", PendingOp::Type::RemoveAll });
}
```

We don't need to protect access to the `HashMap` since it is still only ever accessed by the main thread in `ApplyPending` and `Update`. We need to use `std::scoped_lock` on the first line of `ApplyPending()` too:
```c++
void Coroutines::ApplyPending()
{
	std::scoped_lock lock(mutex);
	
	for (auto& [pair, key, type, async] : pendingOps)
	{
		switch (type)
		{
		case PendingOp::Type::Add:
			async ? asyncCoroutines.Insert(key, pair) : blockCoroutines.Insert(key, pair);
			break;
			
		case PendingOp::Type::Remove:
			// Calling remove with a key that's not in the HashMap won't hurt anything.
			blockCoroutines.Remove(key);
			asyncCoroutines.Remove(key);
			break;
			
		case PendingOp::Type::RemoveAll:
			blockCoroutines.Clear();
			blockCoroutines.Resize(1);
			asyncCoroutines.Clear();
			asyncCoroutines.Resize(1);
			break;
			
		default:;
		}
	}
	pendingOps.Clear();
}
```

Most of the complexity is in `Update()`:
```c++
void Coroutines::Update()
{
	ApplyPending();

	// start all the async coroutines
	Array<std::pair<std::string, std::future<bool>>> futures(asyncCoroutines.Size());
	for (auto& [key, pair] : asyncCoroutines)
	{
		auto& [func, coro] = pair;
		futures.PushBack({ key, std::async(std::launch::async, &Coroutine::Resume, &*coro) });
	}

	// pump all the blocking coroutines
	for (auto& [key, pair] : blockCoroutines)
	{
		auto& [func, coro] = pair;
		if (!coro->Resume())
		{
			Stop(key);
		}
	}

	// join all the async coroutines
	for (auto& [key, future] : futures)
	{
		if (!future.get())
		{
			Stop(key);
		}
	}
}
```

We start all the async coroutines first. This way, they can carry out their work while the main thread is busy looping through the blocking coroutines. We launch them using `std::async` with the `std::launch::async` flag. This ensures that the coroutine _will_ be on its own thread. The default behavior if omitted is whatever the OS deems best, which may not be what we want. I believe that if the user explicitly said that they want this coroutine to be asynchronous, then that's what we'll do.

`std::async` returns a `std::future` which, at some point in the future, will contain the `bool` returned by `Coroutine::Resume()`. Later, when we call `future.get()` this is a blocking call that waits until the thread completes and the `bool` is ready.

## Honorable Mentions
Not included in this write-up is how I count operations to minimize the number of `HashMap` resizes. Basically, on `Start()` I increment a counter, on `Stop()` I decrement it, and on `StopAll()` I set it to zero. In `ApplyPending` I use this counter to figure estimate how big the `HashMap` will become and pre-resize it so that hopefully it won't need to be resized multiple times while applying the pending list. I didn't include it because the code is simple and it's honestly a bit dubious if this improves much since my `HashMap` resizes exponentially and we wouldn't expect to need to resize often in the first place.

Also not mentioned is my `AggregateException` type which I'll gloss over now:
```c++
class AggregateException : public std::exception
{
public:
	Array<std::exception_ptr> exceptions{};
};
```
It's nothing but a collection of other exceptions. The purpose of this is that, when a `Coroutine` throws, we capture the exception into this collection and move on to the next `Coroutine`. That way we guarantee all `Coroutine`s will be run every frame and we only handle errors at the end.

There's also, arguably, a problem with how we're handling asynchronous coroutines. They could be _more_ asynchronous. They're still very much attached to the main thread. The main thread cannot progress until all asynchronous coroutines are in a paused state, and the coroutines cannot resume until the main thread gives them to go-ahead. All asynchronous coroutines still run while the main thread is in `Coroutines::Update()`. It's possible the user might want to make their coroutines behave _completely_ asynchronously.

To achieve this, we can store a `std::shared_future` representing the `Coroutine`'s state, rather than storing a `std::shared_ptr` to the `Coroutine` itself:
```c++
static inline HashMap<std::string, std::shared_future<void>> asyncCoroutines{};
```
Then we add to `asyncCoroutines` like so:
```c++
asyncCoroutines.Insert({ key, std::async(std::launch::async, [coroutine] { while (coroutine->Resume()); }) });
```
We wrap the coroutine into a lambda that will forever keep trying to `Resume()` the `Coroutine`. The lambda will only end when the `Coroutine` is finished. Since we're passing in a `std::shared_ptr<Coroutine>` by value, the `Coroutine` will survive as long as the lambda does.

Then in `Update()` instead of starting them, holding onto their `std::future`s, and joining them, we simply check if they're done.
```c++
for (const auto& [key, pair] : asyncCoroutines)
{
	const auto& [func, future] = pair;
	if (future._Is_ready())
	{
		Stop(key);
	}
}
```
`._Is_ready()` is a Microsoft extension that may or may not be available in whatever implementation of the standard library you may have. It's semantically the same as `.wait_for(std::chrono::nanoseconds(1)) == std::future_status::ready`.

Something that might be convenient about this is that there's only ever one call to `std::async` for every `Coroutine`. Meaning each `Coroutine` is guaranteed to always run on the same thread. That might be useful if some user-code relies on checking the state of the current thread.

There's problems with going this route though. For one, we're introducing countless potential race conditions. Before, async coroutines were limited to only being called within `Update()`, so you could make some safety assurances about the state of the engine. Now nothing is safe. What if the coroutine does something unwanted while the engine happens to be in the middle of a render? Or doing a physics calculation? There's too many aspects of a game engine to make it completely thread safe and doing so would result in a lot of unwanted overhead.

Another problem with this implementation of true asynchronousness is performance. For simplicity's sake, I've used a single `while` loop that just repeatedly checks of the coroutine can `Resume` and whether it is finished. The problem with this is that, when the `Coroutine` isn't doing anything, this creates a busy-wait. The thread will waste precious cycles looping and not doing anything. We could fix this with a simple call to `std::this_thread::sleep_for()`. A more comprehensive solution would be to involve thread pooling.

An idea worth mentioning is a sort of `SelfDestructingThread` type. I've had some ideas on how to implement this for a while but haven't perfected it yet. Basically, you spawn a thread and just let it run and when it's done it destroys itself and all memory associated with it. The main problem with it is I can't decide if there should or shouldn't be a way to get a handle to it or not. Having a handle would be useful if you want to kill the thread early, but it would also partially defeat the purpose of it being "self-destructing".

<sub><sub> Credit goes to <a href="https://blog.panicsoftware.com/your-first-coroutine/" target="_blank">panicsoftware's tutorial</a> which really helped me with understanding C++20's coroutines </sub></sub>
