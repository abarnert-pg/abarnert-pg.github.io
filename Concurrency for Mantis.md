# tl;dr

I think we want to design future Mantis/PGEngine APIs around futures (maybe Boost's, migrating to `std` whenever it finally catches up but that's probably 2023 at the earliest)—but many of them will actually be wrapping either a synchronous API or a callback-based async API, which will also be exposed.

# Synchronous

You want start a battle. Before you can do so, you need to do this pseudocode:

    List<LoadedAssets> PrepBattle() {
        userState = server.GetUserState(user)
        enemyState = server.GetUserState(enemy)
        assetKeys = FindAssetsFor(userState) + FindAssetsFor(enemyState)
        loadedAssets = []
        for assetKey: assetKeys {
            loadedAsset = DownloadAsset(assetKey)
            if assetKey.isShader() {
                loadedAsset = CompileShader(loadedAsset)
            }
            loadedAssets.append(loadedAsset)
        }
        return loadedAssets
    }

But this could be downloading hundreds of files from the DCON server and compiling dozens of shaders. So if you actually do it this way, it could take seconds, during which time you're not rendering any frames or responding to OS events.

There are also huge missed opportunities for optimization. You want to do multiple network downloads concurrently so you don't sit around doing nothing while waiting for the server, and parallelize heavy CPU work like shader compiles so you use all 8 cores instead of just 1, and pipeline as much of that CPU work as possible while there are still later downloads going on, and so on.

# Threads

One option is explicit threads and signaling, as in this pseudocode:

    void PrepBattle(List<LoadedAssets> &loadedAssets, Condition &cv) {
        userState = server.GetUserState(user)
        enemyState = server.GetUserState(enemy)
        assetKeys = FindAssetsFor(userState) + FindAssetsFor(enemyState)
        loadedAssets = []
        loadedAssetsLock = Lock()
        sem = Semaphore()
        downloadThreadPool = ThreadPool(kConcurrentDownloads)
        compileThreadPool = ThreadPool(kConcurrentCompiles)
        for assetKey: assetKeys {
            sem.inc()
		    downloadThreadPool.submit([&]{
		        loadedAsset = downloadAsset(assetKey)
    		    if assetKey.isShader() {
	    	        compileThreadPool.submit([&]{
		                loadedAsset = compileShader(loadedAsset)
		                {
		                    ScopedLock(loadedAssetsLock)
		                    loadedAssets.append(loadedAsset)
		                    sem.dec()
    		            }
	    	        })
		        } else {
		            ScopedLock(loadedAssetsLock)
		            loadedAssets.append(loadedAsset)
		            sem.dec()
    		    }
	    	}
    	}
	    sem.wait()
	    ScopedLock(cv)
	    cv.notify()
    }
	
This is pretty clunky. And I haven't even dealt with errors yet—if you just throw an exception in one of those threaded tasks, it isn't going to reach the top-level function. There are things we could do to improve it, but there are limits… anyway, let's move on.

# Callbacks

One way to solve some of these problems is to make the concurrency internal to the library. There's an `AssetDownloader` object that knows how to download assets in parallel, a `ShaderCompiler` object that knows how to compile shaders in parallel, etc. But then you need some way for each object to tell the client code when it's done without them needing to block their thread or poll periodically. So you take a callback, and call it when you're ready.

Here's an example of the same code written with callbacks.

    void PrepBattle(Function<List<LoadedAsset>(void)> cb) {
        server.GetUserState(user, [&](eUserState) {
            userState = Unpack(eUserState)
            server.GetUserState(enemy, [&](eEnemyState) {
                enemyState = Unpack(eEnemyState)
                assetKeys = FindAssetsFor(userState) + findAssetsFor(enemyState)
                sem = Semaphore()
                loadedAssets = SynchronizedList()
                for assetKey: assetKeys {
                    sem.inc()
                    AssetManager.download(assetKey, [&](eLoadedAsset), {
                        loadedAsset = Unpack(eLoadedAsset)
                        if assetKey.isShader() {
                            ShaderManager.compile(loadedAsset, [&](eCompiled) {
                                compiled = Unpack(eCompiled)
                                loadedAssets.append(compiled)
                                if !sem.dec() { cb(loadedAssets) }
                            })
                        } else {
                            loadedAssets.append(loadedAsset)
                            if !sem.dec() { cb(loadedAssets) }
                        }
                    }
                }
            }
        }
    }

That fixes a lot of problems. But it's even clunkier than threads. Doing 7 things in a row means nesting our code 7 times. (Of course you can refactor that out into named functions, but that adds even more boilerplate and means the related code isn't local anymore.) Dealing with success has become even more complicated (because we have to know, down in each leaf callback, whether this is the end), although dealing with errors is easier (although I've still only done half of it—that `Unpack` function—because the other half takes a lot of boilerplate).

Again, there are ways to improve this, within limits, but let's move on.

## Consistent callback convention

It makes everything a lot better if you use a consistent API for all callback functions.

For example, in Node.js, all async functions take a single callback as the last parameter, and it always takes two parameters: an `Error` (null on success) and a result value (undefined on failure). You can pass callbacks down the chain without having to wrap them. You can programmatically wrap any async function in a future. (In C++, this would probably not be doable even with clever templates, but might be doable with a compile-time code generator tool.) Debugging tools can track lost callbacks (which is a very common problem, and hard to debug without helpers). And so on.

I haven't thought out exactly what we'd want for C++.

# Futures

A future is a wrapper around a value that may not be available yet.

Callbacks turn your control flow inside-out, because instead of just returning values (as you do in synchronous or explicitly-threaded code) you have to pass functions to deal with those values. Futures undo that by letting you go back to returning (future-wrapped) values.

But futures aren't usable if the only thing you can do is block or poll to get the value; otherwise, ultimately you're back to synchronous behavior. The key is that they can be composed in all kinds of ways. I'm guessing nobody would understand if I explained it in terms of monads, so let's just give the API and then show an example.

    class Future<T> {
        T Get(timeout=FOREVER)
        Future<U> Then(Function<T -> U>)
        Future<U> Except(Function<exception_ptr -> U>)
        void Cancel()
    }
    Future<List<T>>   All(List<Future<T>>)
    Future<List<T>>   Any(List<Future<T>>)

You mostly do a lot of `Then` calls—which means you're still writing callbacks, right? Yes, but you're passing futures upward instead of callbacks downward, and that really is better in a lot of ways.

Here's the example:

    Future<List<LoadedAssets>> PrepBattle() {
        return server.GetUserState(user)
            .Then([](userState) {
                return server.GetUserState(enemy).Then([](enemyState) {
                    return FindAssetsFor(userState) + FindAssetsFor(enemyState)
                })
            })
            .Then([](assetKeys) {
                futures = []
                for assetKey: assetKeys {
                    futures.append(AssetManager.download(assetKey).Then([](loadedAsset) {
                        if assetKey.isShader() {
                            return ShaderManager.compile(loadedAsset)
                        } else {
                            return loadedAsset
                        }
                    }))
                }
                return All(futures)
            })
    }

This is still a bit complicated, but not nearly as bad as the raw callback example. Plus, it already takes care of error propagation automatically. (At least if your code is exception-safe so you can propagate errors via exceptions.)

## Promises

It's not that hard to wrap callback-driven APIs in future APIs. For example, let's say we already had a `ShaderManager` that took `success` and `failure` callbacks in `RenderCore`, and we didn't want to rewrite all of `RenderCore` around futures, but we wanted to use it in our futures-based app.

The key is promises, which are the implementation side of futures. I won't give the API here, just an example of using it:

    class WrappedShaderManager : public ShaderManager {
        Future<LoadedAsset> Compile(LoadedAsset source) {
            promise = Promise<LoadedAsset>()
            Compile(
                source, 
                [](LoadedAsset compiled) { promise.Fulfill(compiled) },
                [](exception_ptr ex) { promise.Reject(ex); })
            return FromPromise(promise)
        }
    }

… and now (as long as we remember to always wrap the compiler manager at the boundary between `RenderCore` and our application code), we have the futures API.

(Realistically you probably want to encapsulate and delegate instead of inheriting.)

## Executors

It's even easier to wrap synchronous APIs in future-driven APIs, by using executors. Executors are just objects that take a function and return a future. Common types of executors wrap thread pools (with or without bounded queues), new-thread-per-task, run-in-main-thread… basically anything familiar from Grand Central Dispatch, and more, can be written pretty easily as an executor. (See the Java standard library for a zillion examples.)

As long as there's a good owner for an executor—the chunk of code that prepares for starting a battle, or a wrapper around the synchronous GameServer interface that makes it asynchronous, or whatever—you just submit synchronous tasks and that's all there is to it. For example, let's say our server object above was synchronous. We could just do this:

    class ThreadedGameServer : public GameServer {
        executor = GetDefaultWebServiceExecutor()
        Future<UserState> GetUserState(user) {
            promise = Promise<UserState>()
            executor.Submit([&]{
                try {
                    promise.Fulfill(super::GetUserState(user))
                } catch (exception e) {
                    promise.Reject(e)
                }})
            return FromPromise(promise)
        }
    }

… and now we've got an asynchronous server object, with a futures-based version of the same API.

It's also often useful to use executors directly instead of in wrappers, especially for handing off tasks between executors. (Some futures APIs let you pass an executor directly in the `Then` call, instead of having to `Then` a `Submit`.)

However, it's worth noting that in some cases, internal concurrency is better. For example, an HTTP client that manages its own thread pool can do things like try to schedule each request on a thread that's currently talking to the same host, to maximize the use of persistent connections; there's no way a generic executor can know that.

## Control flow

Most of the "control flow" above is just "do this, then that, then the other". What if you need to loop or switch? In some cases, the loop or switch can be pulled out of the future—as in the example above, where I created a list of futures in a loop and then combined them with `All`. For common patterns like map/reduce/filter it can be even nicer. 

But anything that doesn't fit those, you basically need functions like `While([]{ condition }, []{ body })`, `ForEach(range, [](value) { body })`, `ForEachParallel(range, [](value) { body })`, `IfElse(condition, []{ ifbody }. []{ elsebody })`, etc.—and they look even clunkier when you remember that most of those bodies are going to have further lambdas inside them. It ends up a lot like template metaprogramming, where if you do anything nontrivial it feels like you're writing code in a horribly verbose and limited dialect of Lisp rather than in C++.

Still, it's not as bad as with explicit callbacks.

# Awaitables

Let's just jump right into an example here:

    List<LoadedAssets> PrepBattle() async {
        userState = await server.GetUserState(user)
        enemyState = await server.GetUserState(enemy)
        assetKeys = FindAssetsFor(userState) + FindAssetsFor(enemyState)
        loadedAssets = []
        for assetKey: assetKeys {
            loadedAsset = await downloadAsset(assetKey)
            if assetKey.isShader() {
                loadedAsset = await compileShader(loadedAsset)
            }
            loadedAssets.append(loadedAsset)
        }
        return loadedAssets
    }

This looks identical to the synchronous code, except for those `async` and `await` keywords. And it _works_ identical to the synchronous code, except that at each `await` it's actually suspending operation until the called function has a value for us.

It's not quite as magical as that (except in languages with first-class continuations, like Alice ML, where it really is). Your function is a coroutine, and the awaitable is the result of another coroutine, and what `await` does is tell your scheduler to switch to the awaitable's coroutine and remember to resume you later at some point after the awaitable is ready. This is hard to explain, but if you look over some example code for Python's `asyncio`, `trio`, or `curio`, or probably for JavaScript, it's not that hard to grasp.

This makes `await` contagious: only an `async` coroutine can `await`, and that means it can only be called by another coroutine, and so on up the chain (until, usually, you've got some code in a regular function that's `Submit`ting a coroutine to an executor).

In Python and JS, coroutines are built on top of futures, and they're exposed to the user. You can `await` a future, which means you can `await` running a task on an executor (even if that's a thread-pool executor that doesn't know from coroutines). Or grab a bunch of awaitables and `All` or `Any` them into a single thing you can `await`. (In C#, awaitables don't directly expose futures, but there are migratable coroutines and thread-pool coroutine schedulers, so you don't need them as often. That might be even nicer for games, but even harder to implement for C++.)

This is great if you have a lot of code that makes sense to run in a single-threaded coroutine scheduler, as well as some code that doesn't. Like a server, where the core of the server is a reactor that needs to handle thousands of sockets (you can easily run one coroutine per socket). I'm not sure it's as useful in a game, but it might be.

`await` is really just syntactic sugar, and without sugar it still doesn't look too bad:

    userState = Await([]{ server.GetUserState(user) })
    enemyState = Await([]{ server.GetUserState(enemy) })
    assetKeys = FindAssetsFor(userState) + FindAssetsFor(enemyState)
    loadedAssets = []
    for assetKey: assetKeys {
        loadedAsset = Await([]{ downloadAsset(assetKey) })
        if assetKey.isShader() {
            loadedAsset = Await([]{ compileShader(loadedAsset) })
        }
        loadedAssets.append(loadedAsset)
    }
    return loadedAssets

… but then the compiler can't help us catch common and painful-to-debug mistakes like `await`ing in a non-coroutine. Which really is a big deal—the reason Python has `await` even though you can do the same thing with `yield from` is that most people found `asyncio` too much of a bug magnet to use in production code without compiler support.

It may, however, be possible to explicitly pass around a "context" object and actually get the same benefits. It's more boilerplate, but not much more. I don't know how nice this is in practice.

## Greenlets

You can do the same basic thing as `await` without `await`, by putting all of the coroutine-related stuff inside each asynchronous function instead of in the client, and then you don't need any language support. The big problem is that, without anything marking where the client code can block, it's a lot easier to accidentally create races and similar threading-style problems. Still, people have written plenty of serious production code with libraries like Python's `gevent` and even C++'s `Boost.Asio`.

## Fibers

Fibers are basically coroutines, but with an API that looks like threads. Just as with greenlets, the key to making it work is that every asynchronous function has to know how to block and unblock itself, so the client code doesn't have to worry about it. Since it's the same thing with a different API, it has the same problem with races and so on—but because that API looks like threads, people who know how to deal with thread synchronization know how to deal with fiber synchronization. (Although there can be performance costs for that—e.g., grabbing a lock around some code that couldn't possibly switch is wasted cost.)

## Multiplex fibers

Once you bite the bullet and just require client code to lock and signal the same way as in preemptively threaded code, then you can easily create fibers that can migrate between threads on a thread pool. And you can even throw CPU-bound tasks at them, as long as you're aware that you're blocking up one of the pool's threads, or willing to add explicit `YieldToScheduler` calls every so often. They're basically just like threads, except that instead of only being able to have a few dozen of them, you can have thousands of them as long as only a few dozen are CPU-bound. (And some of the fiber libraries out there can do really clever things, like improving NUMA performance by scheduling fibers that share data on the same thread almost as well as the OS can schedule threads that share data on the same core.)

# Language support

As of early 2020, the language supports none of this:

 - Futures do exist—but in a useless, stripped-down implementation.
   - C++20 still has the useless version from C++11.
   - TS1 (post-C++17 experimental) is better, but not really available, and a dead end.
   - TS2 (post-C++20 experimental) isn't even at the published draft stage yet.
 - Coroutines will probably be in C++20, but there are only experimental implementations so far. And `await` seems to have been bumped to a new post-20 TS.
 - Fibers and greenlets do not have live proposals at all, only withdrawn ones from 5+ years ago.

However, all of these things (even `await`, but with clunkier non-sugared syntax) can be built without language support, as long as you only care about working on iOS/Android/Mac/Windows/Linux/*BSD rather than on every possible C++ implementation. And there are third-party libraries that do it.

In fact, Boost has all of that in one place, and it all works together (and with standard library features), and whatever ends up in the stdlib will probably be closely aligned with Boost (as it usually is). But that's a pretty big one place. And using Boost—at least the parts that need to be compiled and linked, especially when you're cross-compiling (as we are)—can be a pain. So we may not want to use it.

There are other implementations out there if we don't want (or can't use) Boost.

# (Tentative) conclusions

I don't think anything around coroutines/fibers/etc. is likely to be that helpful to games, at least without language-native `await` support on top of it and libraries that make use of it, which is probably not coming any time soon. (Unity has supported C# `await` since late 2017; are games using it?)

However, futures and executors are a useful way to tie everything together. If using Boost is acceptable, or I can find a smaller third-party library that works for us, I think it is worth designing around that.

Some of our code is inherently callback-driven, like much of `RenderCore`, and that code should still expose a callback-driven API. In some cases there should be a futures wrapper around it, but sometimes nobody will ever need it. (And this means we don't need to go wrapping and/or rewriting everything in `RenderCore` and `PGEngine` just in case it might be needed; we can wrap things as the need arises.)

Some code inherently needs internal concurrency but isn't callback-driven. That code can expose futures only.

But a lot of code will work fine with external concurrency. So we might as well just write synchronous APIs (which are always simpler). In some cases it will be worth also adding a futures-and-executor wrapper; in other cases, letting the client manage the executors makes more sense. If you're not sure, you can always not write the wrapper now, and add it later if needed.

Which one is HTTP? It really depends on (and helps determine) which backend we choose. If we find a great HTTP library that cleverly takes advantage of managing its own thread pool, we'll want internal concurrency. If the best fit is inherently synchronous, we might as well expose the sync interface plus an executor wrapper.

We _might_ want to consider a design inspired by the "universal model for asynchronous operations" in N3747 (which drives the standardization committee, and Boost). The idea is that rather than choosing between callbacks and futures or wrapping one around the other, you have a callback API that supports futures on demand. Instead of returning `void` you return an `AsyncResult`, which is normally an empty object, but if you pass a special marker value instead of a callback function it's a future. But I don't think we need this. We don't need to interoperate with every possible third-party library the way something intended to be universally general like Boost does. We don't need to eliminate nanosecond costs so our API can be used in apps that handle 10000 sockets with a coroutine per socket. And so on. And it does add boilerplate to every function. But I'll look at how much of a hassle it is; if it really isn't that bad, maybe it's worth doing.

We've got a lot of code written around Grand Central Dispatch today. So we should provide an (optional, so we can make appportable and libdispatch optional) futures-and-executors wrapper to interoperate with GCD.

We also need an executor that wraps our new frame loop design (whatever that ends up being).

A consistent callback API is important whether we use futures or stick with pure callbacks.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MjU2MjUwMDksOTU3ODcyNTY0XX0=
-->