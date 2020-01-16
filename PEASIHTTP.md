Killing the ASIHTTP interface is not too hard in ToW, and most of the trouble is actually inside PGEngine itself.

The PEASIHTTP interface still needs to be reimplemented on top of a different backend, but after stripping it down, it doesn't seem too hard.

# PGEngine

## Interface

There are two key classes: `PEASIHTTPRequestAgent` and `PESecureASIHTTPRequestHeader`. The latter just adds request signing around the former, and needs little if any change. There are also a number of helper classes that are part of the interface, but most of them are trivial.

`PEASIHTTPRequestAgent` has a whole slew of methods for starting a request. However, most of them are deprecated—and the ones that can't be removed are just thin wrappers around the others.

The remaining methods have a clean, simple interface, except that they expose `ASIHTTPRequest` directly to clients, and that class is chock full of complicated behavior.

Fortunately, almost nobody uses any of that behavior. Having a class named `ASIHTTPRequest`, but that's just a dumb bag of read-only properties, almost works out of the box. (Of course if you use ASIHTTP as the backend, this name conflicts; I fixed that by renaming the backend class to `OldASIHTTP`.)

It's also probably worth removing a bunch of the other properties on the agent that nobody uses.

### `PEASIHTTPRequestAgent` constructor methods

There's one underlying constructor `agentWithParameters`, which takes a special agent-parameter object. (It's a bit confusing that, throughout the interface, "parameters" sometimes means "things about this HTTP request" like the URL and authentication and so on, and sometimes means "query-string or form-data variables". In this case, we're talking about the former.)

There are three "designated API methods" on top of this, one to construct an agent for an arbitrary HTTP method, the others for posting a body, and posting files, all with a bunch of options that are rarely but occasionally needed.

There are two convenience methods for simple GET and POST requests with nothing but a URL and parameters. These are used in a few places in ToW.

There are a slew of additional convenience methods for GET, POST, PUT, and DELETE with different combinations of options, which are all deprecated. For the main class, none are used in ToW; one is used one time inside PGEngine, in `PGFacebookManager`, which can be easily fixed. So, the deprecated methods all go away. For the `Secure` class, there are more places that use the deprecated methods. That could be fixed, but there's not much harm in leaving them.

### `PESecureASIHTTPRequestAgent` constructor methods

This subclass doesn't expose anything but overrides of all of the constructor methods and some additional parameters.

The same constructor variants are deprecated, but some of them are actually used in the code. If they're not removed but the superclass ones are, a couple of them may need to be changed to call the designated API methods instead of to call the (removed) superclass methods. But that's it; the rest just works.

### `bandwidthMonitor`

ASIHTTP has a hook, `bandwidthMonitor`, for clients to monitor bandwidth, and it's used by ToW.

The interface is pretty simple. There's just a class property on `ASIHTTPRequest` (which could probably move to `PEASIHTTPAgent`) to install the delegate, and then a couple of the delegate's methods have to get called by the backend to update the bandwidth usage. The implementation might be painful with some backends, however. Maybe there's a better solution for doing this, or maybe we can scrap it?

At any rate, for now, there's a proxy on `PEASIHTTPAgent` that delegates from `ASIHTTPRequest` to the specified delegate. I'm not yet sure whether that interface will be stable with a different backend.

### `ASIHTTPRequest`

The `ASIHTTPRequest` class is the main interface to the underlying ASIHTTP library. It's a tremendously complicated thing. It can represent a request that hasn't been sent yet, a request with a completed response, a request with an open response that exposes streams to read from, etc., and it has even more properties and methods than you'd expect from that complexity.

Fortunately, almost all of the code in ToW and PGEngine doesn't use any of that behavior; it only ever gets an `ASIHTTPRequest` that represents a completed request, calls one class method and no instance methods, reads 14 simple properties ("simple" meaning things like the request URL or response content data, as opposed to things like the socket's read stream or the SSL context), and writes nothing. So, we can create a new class with that name that has just 14 simple read-only properties and one class method, and none of the complexity of ASIHTTP leaks to client code.

It would be nice to rename this to something that didn't start with `ASIHTTP`, but that seems about as urgent as renaming the wrapper library and its classes to something that doesn't include `ASIHTTP`. Yes, it's a bit confusing, and the refactor could be trivially automated, but it's a huge commit, so I'm not doing it for this prototype.

There are some places that do use more of `ASIHTTPRequest` in trivial and easily fixable ways, and at least one place that does so more drastically; they'll be dealt with below.

### `ASIFormDataRequest`

`ASIFormDataRequest` is another part of ASIHTTP that's exposed to a few clients of PEASI—but none of them actually need it at all, so that's easy to fix. (It is needed by PEASI internally, but that's fine; it's not exposed to clients.)

## Higher-level wrappers

Most of the higher-level wrappers around PEASI in PGEngine work as-is, or need only minor tweaks to work with the bag-of-props request object. (For example, one wrapper tries to clear out the request, which we can't do—but since we're about to release the request anyway, and tell the agent to clear its data, we don't need to.)

### `AssetRequest`

`AssetRequest` is a more serious problem. It uses ASIHTTP directly, and there are three reasons it can't just be trivially converted.

#### `kShouldDownloadAssetDirectlyToDisk`

If split test option `kShouldDownloadAssetDirectlyToDisk` is enabled, it asks ASIHTTP to stream the asset directly to a disk, then reads the file into memory, instead of asking ASIHTTP to stream the file into memory, then writing it to disk.

I'm about 95% sure this can be scrapped.

The usual advantages of doing this don't exist in `AssetManager`. Streaming to file can be simpler, but in this case we have to implement two methods both ways instead of just one way, so it's more complicated. It can also be more efficient (if the framework can ask the OS to go directly from network buffers to the filesystem without passing anything through userland), but not if you need to read the whole thing into memory at the end anyway.

The one advantage it does provide is that, if you also enable the zipfile-asset option and the multipart-zipfile-asset option, the stream-to-file option can handle multipart zipfiles a little better. But not significantly better. And in fact, the way we'd really want to handle this is to assemble the multipart zipfile without writing the individual parts to disk at all. (It would be even better if we could parse it iteratively without ever having to assemble the whole thing, but sadly, the way zipfiles work, you have to start at the end of the archive.) At any rate, the other code path for multipart zipfiles without stream-to-disk works just fine.

Plus, ToW doesn't use these options; they're Dragons-specific. And I don't think a new game would need them if ToW doesn't.

So, the `kShouldDownloadAssetDirectlyToDisk` option can just be scrapped.

#### `ASIDoNotWriteToCacheCachePolicy`

`AssetRequest` uses most of the same options as PEASIHTTP, except for the cache policy.

I believe that this option is used because DCON has its own cache, so you should never be downloading the same asset file twice, so storing one in the HTTP cache is just wasting cache space. Which seems like behavior worth preserving.

This could mean extending the designated API with a variant to the simple request (no post data or files) method, meaning we now have 4 methods instead of 3. But maybe the asset manager can just use the agent parameters API?

At any rate, leaving this out for the prototype probably means that DCON assets get double-cached, wasting disk space or pushing other stuff out of the cache, but as far as I can tell, it doesn't actually break anything (and it shouldn't).

#### `startSynchronous`

PEASIHTTP only uses the async mechanism in ASIHTTP, where you set up a request, set its callbacks, and call `startAsynchronous`. But `AssetRequest` uses the sync mechanism, where you set up a request, call `startSynchronous`, and then process the results inline.

This allows the asset manager to run its own queue of synchronous download requests, instead of sharing the implicit async pool with all of the other requests. There might be a good reason for this—most of our requests involve sending a small amount of data and receiving a small response, but asset downloads involve sending no data and receiving a comparatively large response. A really clever backend might be able to schedule immediate vs. bulk data automatically, but most can't; keeping them separate might have benefits like, e.g., a slow connection doesn't introduce as much lag on immediate requests because, while they are sharing a network with the bulk data, they aren't queuing up behind bulk requests. However, we don't actually know there is such an issue, or that this is a working solution, that's just a guess.

While we could add a similar API to the PEASIHTTP interface, this may not be a good idea. Most backends don't have sync and async APIs that look nearly identical like ASIHTTP's.

For the prototype, I just fake sync on top of running async and blocking on the callbacks. There are a few annoying threading issues that come up, but I think I got it all working. If it turns out that we can just throw asset downloads into the same async queue as everything else, we can later rewrite the asset manager to sync at a higher level. If it turns out that we can't, then we have just a small amount of prototype code to throw out instead of a whole complex mechanism that's infected the API and constrained our choice of backends.

Meanwhile, the flag `shouldUseASIHTTPRequest` should probably be renamed to something about PEASIHTTP instead, but that's not a big deal for now.

#### ASIHTTP errors

In at least one place, if we've got a download error, we check whether it's a timeout error by using `NetworkRequestErrorDomain` (which is exposed by PEASIHTTP) and error code `ASIRequestTimedOutErrorType` (which isn't). Also, in many cases the error will be a `PEASIHTTPRequestAgentErrorDomain` with a code equal to the HTTP status code and the ASIHTTP error as the underlying error inside that.

We don't want all of the ASIHTTP error types and codes (and, for that matter, the Cocoa Touch ones that it sometimes uses) to be part of our interface. What I've done instead is to add methods to ask PEASI whether an error is a timeout, etc. This may not be the best solution for the final version, but it's fine for now.

# ToW

All code either:

 * Uses higher-level wrappers like `WebServiceInterface` and `AssetManager`
 * Uses PEASIHTTP without even looking at `ASIHTTPRequest`
 * Uses PEASIHTTP but only looks at few of readonly props on `ASIHTTPRequest`

… so it just works.

It does use some deprecated constructors from the `Secure` agent, but none from the regular agent. The deprecated methods would be pretty easy to change, but it would be tedious, and give us a huge commit, for little benefit.

`ASIHTTPRequestBandwidthTracker` is a `bandwidthMonitor` delegate; I changed it from installing itself onto ASIHTTP to PEASIHTTP.

# Dragons

From a quick test, Dragons seems to have more code that doesn't just work out of the box. Maybe not that much, but I didn't want to do more than a quick test. It also uses the deprecated methods a lot more.

# Next steps

So we've got a clean PEASIHTTP interface that ToW can use without seeing any of the ASIHTTP under the covers, and it works in ToW on at least iOS and Mac (still testing Android). What next?

## Interface

The PEASIHTTP interface isn't terrible, and it's already used all over the place by ToW.

But it is ObjC, and uses some not-quite-trivial apportable types like `NSURL`, and we may want future games to use a C++ (or maybe even C?) interface instead.

So, we probably want a C++ interface (let's call it `pgengine::HTTPAgent` for the sake of discussion) that wraps ASIHTTP in the same way as PEASIHTTP. I've put a bit of work into this, enough to verify that it looks feasible but not enough to actually use it.

Then we may want to rewrite the PEASIHTTP ObjC API as a thin wrapper around the C++ API, instead of as a parallel wrapper to the same backend.

We may want to change other parts of PGEngine to use the C++ interface. But as long as those parts are exposing ObjC interfaces themselves (and we're not sure exactly which ones are worth keeping vs. rewriting vs. scrapping), this doesn't seem too urgent.

## Backend

Right now, the backend to PEASI is just a shim in front of ASIHTTP, so obviously the behavior is no better than before. In fact, in a few places, it's a little worse, like `AssetManager` double-caching, etc. The plan is to make it a (maybe not quite as thin) wrapper around a different backend.

What we need from the backend as far as API is pretty minimal. However, we do have some heavy demands about behavior. It has to either let us drive its threading or do this threading in a way that's compatible with what we want. It has to handle things like SSL certs in the same way Apportable does on Android despite not using Apportable. And so on. Most of these things are things that ASIHTTP actually doesn't quite get right (especially on Android), which is one of the reasons we want to replace it… but replacing it with something that offers very different behavior instead of a better but not too different version of the same behavior (for the subset of behavior we want) might require lots of disruption.

Given how thin our API demands are, and how much of our behavior demands are "do the same right thing that Apportable does for us mostly correctly", it's possible that using platform-specific code on each platform is the best answer. On the other hand, there are obvious advantages to cross-platform libraries. For example, the Android-specific `Volley` library (which is recommended by Android docs over what comes in the stdlib) automatically takes care of things that we don't want to worry about, like asking the OS about how to balance memory and disk cache appropriately for the specific device; a cross-platform library may well not do that. But it may do it in a way that we can't transparently wrap up to look the same as caching on iOS with an iOS-specific library. Is it more important to be as platform-optimized for each platform (or at least the two key ones) as possible, or as similar between platforms as possible? I'm not sure yet.

I built a few quick&dirty proof-of-concept backends just to verify that it isn't harder than it looks. One uses `NSURLSession` directly. Which is clunky, doesn't seem to work out of the box with Apportable on Android, breaks a bunch of things (including any plain HTTP instead of HTTPS…), fills the logs with spam about TIC… but it sort of works. Then I did libcurl in single-threaded mode with a queue in front of it, which breaks cookies and HTTPS verification, and sometimes deadlocks the game… but it sort of works. I don't think there are any big hurdles I wasn't anticipating. So, that's enough PoCs; time to try out a serious option in more detail (or to build the C++ interface, then come back to this).

### Multiple/swappable backends

Do we want to make sure it's easy to swap in different backends, or maybe even include multiple swappable backends as part of PGEngine?

If we end up using platform-specific backends, of course there's no question here. But if not, I don't see much benefit, at least beyond the initial development stage. Everyone is always going to use the one "main" backend, and the other ones will bitrot to the point where they require too much work to be worth even testing.

### DCON

Do we want `AssetManager` to use the same HTTP library we expose to the client code?

PEASI is clearly designed around handling interactive web service requests well, and we're looking for a backend that can do that even better than ASIHTTP. Bulk downloading of large files is very different from interactive web service requests.

For example, look at the Android docs: they suggest two different third-party Android libraries, because `Volley`, the best library for web service/RPC requests, is "not suitable" for large downloads (although I believe our existing code—even with the `kShouldDownloadAssetDirectlyToDisk` option—is just as bad…), but `DownloadManager` doesn't do key things like queuing, prioritization, cancellation, etc. flexibly enough for interactive use.

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2NjAxOTUzMzddfQ==
-->