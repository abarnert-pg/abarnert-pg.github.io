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

The caller could just call this function and wait many seconds for it to return, but the game will get killed for being non responsive.

Errors are handled properly (at least if all errors are propagated by exceptions and all objects are exception-safe).

# Threads

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

The caller creates a list and a condition variable, then spawns a thread that calls this function or dispatches it into a thread pool or whatever. When it's ready for the results, it can poll the cv once per frame or whatever. (If you just spawned a thread, you could also poll-join the thread instead of using a cv.)

This allows it to do other things (like rendering frames and handling input) while the work is happening. And it should also be an order of magnitude or so faster.

I left out most of the error handling, because it's already complicated enough as-is.

# Callbacks

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

The caller calls this function passing a lambda as a callback. It has to be organized in such a way that the battle starts after all the relevant callbacks are called.

It should be slightly more efficient than the threaded solution, but not enough to likely matter.

I left out about half the error handling.

Note that this requires all asynchronous tasks, like that `server.GetUserState`, to handle concurrency internally, and to expose it via callbacks.

# Futures

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

The caller calls this function and gets back a future wrapping a list. It can attach a `Then` callback to the future if it's organized around callbacks (or futures), or it can poll the future once per frame or whenever if it isn't.

This is still nearly as efficient as the callback version, although there is a tiny bit of overhead, but again not enough to likely matter.

Errors are handled properly again.

While we're using internally-concurrent objects here, and they have future interfaces, it's trivial to use synchronous objects via an executor. And it's almost as easy (although there's more boilerplate than you'd like—maybe this could be auto-generated, but I'm not sure) to wrap up object+executor in an internally-concurrent object, or to wrap up internally-concurrent objects with callback APIs.

# Awaitables

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

The caller `await`s this function instead of calling it normally, and then when it resumes, it's got the list. Of course this requires the caller to itself be an async coroutine. `await` is contagious—it has to propagate upward until someone explicitly spawns a task into a runloop and gets back a future (or daemonizes a task that's run for pure side-effect purposes with no result or signaling).

This can be even more efficient than the above cases for things like massively-concurrent networking, but for CPU-bound things like compiling a bunch of shaders or scripts it's just adding a bit of overhead to an executor and futures. Either way, it's unlikely to matter for a game. (Games don't normally need to handle thousands of concurrent connections, for example.)

We're requiring every object to use async coroutines. However, it's trivial to await futures, which means that wrapping synchronous and callback-driven APIs is no harder than it is with futures.


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTM5NjU3MzA5N119
-->