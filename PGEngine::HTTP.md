# `PEASIHTTP`

Currently, the main HTTP library in PGEngine is PEASIHTTP, a wrapper around the open source library ASIHTTP. You use it like this:

    PEASIHTTPRequestAgent *agent = 
        [PEASIHTTPRequestAgent getWithURL:url
                               parameters:nil
                                  success:^(PEASIHTTPRequestAgent *agent) {
            NSLog(@"got successful HTTP %d with %u bytes", agent.request.responseStatusCode, agent.request.contentLength);
        }
                                  failure:^(PEASIHTTPRequestAgent *agent) {
            NSLog(@"got failed HTTP %d with error %@", agent.request.responseStatusCode, agent.request.error);
        }];

There are a handful of similar methods: one similarly short one for post, four with a slew of extra args for get/post body/post files/arbitrary request, and one for explicitly building a structure full of args and making a request with that. But the basic idea is the same for all of them.

Notice that, even though this is a class method constructor, there's almost never any reason to actually use the constructed object that's returned. The same object is passed into the callbacks, which is where it's actually used. And even there, the only thing you want to do with it is get the request object. (There is one exception to all of this, in `AssetManager`—but it relies on peeking under the covers and using ASIHTTP features.)

# `PGEngine::HTTP`

Here's how you use the new C++ API to do the same thing:

    PGEngine::HTTP::Get(url, {}, [](response) {
        LOG("got successful HTTP %d with %u bytes", response.statusCode, response.rawContent.size());
    }, [](response) {
        LOG("got failed HTTP %d with error %s", response.statusCode, response.error.what());
    });

# API details

The API is intentionally as limited as possible—like PEASI, but more so (and without a backdoor to get to ASIHTTP underneath). Client code has no control over cookies, threading, cert stores, etc.; no access to the socket streams; you just make a request and get a callback with a response when it's done. Much like PEASI, but even more stripped down.

    // TODO: More needed here? Should it be an exception that you can rethrow? Or contain one?
    struct Error {
        string what() const;
        bool   isTimeout() const;
        bool   isNetworkDown() const;
    };

    struct Request {
        vector<char>        content;
        map<string, string> headers;
        string              method;
        string              url;
    };

    struct Response {
        vector<char>        content; // decompressed rawContent
        Error               error;
        map<string, string> headers;
        string              path; // only with GetFile
        vector<char>        rawContent; // empty with GetFile
        Request             request;
        int                 statusCode;
        string              statusMessage;
        string              text; // decompressed and decoded
        string              url; // this is the final URL after any redirects
    };

    void PGEngine::HTTP::Get(
        string url, 
        map<string, string> parameters,
        function<void(const Response &)> success, 
        function<void(const Response &)> failure,
        bool sign=false, 
        bool verify=false,
        double timeoutSeconds=HTTP::kDefaultTimeoutSeconds,
        uint retryCount=HTTP::kDefaultRetryCount,
        double retryDelayOn500=HTTP::kRetryDelayOn500,
        double retryDelayOn500Multiplier=HTTP::kRetryDelayOn500Multiplier,
        bool cache=true);

    void PGEngine::HTTP::GetFile(
        string url, 
        string filename,
        const map<string, string> &parameters,
        function<void(const Response &)> success, 
        function<void(const Response &)> failure,
        etc.);

    void PGEngine::HTTP::Post(
        string url, 
        const map<string, string> &parameters,
        function<void(const Response &)> success, 
        function<void(const Response &)> failure,
        etc.);

    void PGEngine::HTTP::Post(
        string url, 
        const vector<char> &body,
        function<void(const Response &)> success, 
        function<void(const Response &)> failure,
        etc.);

    void PGEngine::HTTP::PostFiles(
        string url, 
        const map<string, string> &files,
        function<void(const Response &)> success, 
        function<void(const Response &)> failure,
        etc.);

    void PGEngine::HTTP::Request(
        string method,
        string url, 
        const map<string, string> &parameters,
        function<void(const Response &)> success, 
        function<void(const Response &)> failure,
        etc.);

## TO(NOT)DO

This should cover all the uses of the param-structure API, not just the simpler API, that are actually used in PGEngine/Dragons/ToW. But it does leave out a few things.

There are params today for `serverPort` and `compressBody`. Neither is used in PGEngine or ToW code. We always use the port from the URL (or from the service base URL, when using the utilities that build a URL from a service and an endpoint name), and we always use compress (which actually means compress if the server can handle it, fall back if it can't—less efficient for servers that can't handle it, but we don't talk to any).

There are a bunch of response attributes that are either redundant (e.g., `contentLength`) or never used (and not even correct in the case of cookies…), and also a bunch of them are available directly on the agent object as well as on its request property… but that's all unneeded.

Using jQuery-style callback/errback, but not putting it at the end (because all of the usually-defaulted arguments have to come after the mandatory ones because C++ sucks), looks weird to anyone who's used to the jQuery style. But I don't think it's an actual problem here.

### Too many damn params

Requiring the user to pass a whole slew of extra params when they want to change just one of them is annoying. But that's in inherent in C++ sucking (and ObjC too). All of the tricks to simulate named arguments are either annoying to use, fragile, or ridiculously complicated in implementation (look at the Boost version if you want to have nightmares about macros and templates fighting over a post-apocalyptic Earth where humans can only survive in deep caves). We could use one if people really hate this API, but otherwise it's not worth it.

There are other things we could do to at least reduce the number:

 - In many cases, we pass the same function for `success` and `failure`. Often by writing the exact same block twice. We could have a special value for `failure` (and even make it the default) that means "same as `success`". But if the callback is long enough that repeating it bothers you, it's probably long enough that naming it and passing it by name won't bother you.

 - We could have a separate function for the "Secure" versions of each, as in PEASI, where the non-Secure version doesn't take the `sign` and `verify` params. But I think that just makes things bigger instead of smaller.

 - We could replace `sign` and `verify` with a single `secure` param that takes an enum with values `NoSign | SignOnly | SignAndVerify`.

 - We could replace the three timeout-related params with a single one that takes an enum with values `Default | NoRetry | WaitForeverNoRetry | InfiniteRetrySlowBackoff`. I think those are the only four combinations of values used anywhere; there might be one or two more. That's certainly more readable as well as less verbose. The only problem is that, just because Dragons and ToW have no use for any other combinations of these values doesn't mean no future game will.

At any rate, all of these seem like possibly good ideas, but it's probably not worth over-engineering this until we see whether people are annoyed by the existing API, whether they have uses for other values that ToW didn't need, etc.

## ObjC API

The existing PEASIHTTP API can be rewritten as a wrapper around the C++ API for legacy code.

If we want a non-legacy ObjC API, the simplest option is to just wrap each C++ function in a C function with the prefix `PEHTTP` that uses `NSData`, `NSString`, etc. values, and ObjC objects rather than structs for the results, passed to blocks rather than functions, like this:

    void PEHTTPGet(
        NSString *url, 
        NSDictionary *parameters,
        ^(PEHTTPResponse *) success,
        ^(PEHTTPResponse *) failure,
        BOOL sign, 
        BOOL verify,
        NSTimeInterval timeout,
        uint retryCount,
        NSTimeInterval retryDelayOn500,
        NSTimeInterval retryDelayOn500Multiplier,
        BOOL cache);

The obvious problem here is that ObjC doesn't allow default values or even overloading on functions; effectively you get just C89 syntax. Another option is to convert the C++ functions into class methods of a empty, never-constructed ObjC class and then we can use overloads, maybe three of them per C++ method:

    +[PEHTTP getURL:(NSString *)url
         parameters:(NSDictionary *)parameters
            success:(^(PEHTTPResponse *))success
            failure:(^(PEHTTPResponse *))failure];
            
    +[PEHTTP getURL:(NSString *)url
         parameters:(NSDictionary *)parameters
            success:(^(PEHTTPResponse *))success
            failure:(^(PEHTTPResponse *))failure
               sign:(BOOL)sign
             verify:(BOOL)verify];

    +[PEHTTP getURL:(NSString *)url
         parameters:(NSDictionary *)parameters
            success:(^(PEHTTPResponse *))success
            failure:(^(PEHTTPResponse *))failure
               sign:(BOOL)sign
             verify:(BOOL)verify               
            timeout:(NSTimeInterval)timeout
         retryCount:(uint)retryCount
    retryDelayOn500:(NSTimeInterval)retryDelayOn500
         multiplier:(NSTimeInterval)retryDelayOn500Multiplier
              cache:(BOOL)cache];

We could even use the builder-pattern workaround for named-and-defaulted parameters, which is less horrible and more idiomatic in ObjC than in C++, but I'm not going to write that out here.

It depends on how much we care about having a good non-legacy ObjC API. I think for the initial version, we can just not have one at all, and see what the demand is later.

# Higher-level interfaces

PGEngine contains a number of higher-level interfaces to PEASI, like `WebServiceInterface`. These can remain unchanged. Initially they'll just use the legacy PEASI wrapper. If we want to change them significantly in the future (e.g., to offer a C++ API, or to add a new feature), we can switch them to using the C++ API as needed.

`AssetManager` is the one big exception to this, but I've got a whole other doc on that.

