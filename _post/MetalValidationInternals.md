---


---

<p>Metal validation is actually pretty configurable, even if the functions to configure it are all private.</p>
<p>Every Metal validation error or warning is handled by calling a function called <code>MTLReportFailure</code>.</p>
<p>And Metal has a builtin hooking mechanism for this function, <code>MTLSetReportFailureBlock</code>. If you call this function to install a report failure block, <code>MTLReportFailure</code> skips all the normal stuff and instead calls your block.</p>
<h2 id="reportfailureblock"><code>ReportFailureBlock</code></h2>
<p>Here’s how to do the same thing as the default, except using <code>NSLog</code> instead of writing to <code>stderr</code>:</p>
<pre><code>MTLSetReportFailureBlock(^(int64_t type, const char *func, int32_t line, NSString *msg) {
    NSLog(@"%s:%d: failed assertion `%@'", func, line, msg);
    if (type == 1) {
        abort();
    }
});
</code></pre>
<p><code>msg</code> is the string you’re usually interested in, like <strong>missing vertexDescriptor buffer binding at index 1.</strong></p>
<p><code>function</code> and <code>line</code> (or maybe it’s an offset, or whatever) aren’t all that useful—it’s not the actual Metal method that you called, but the validation function under the covers, like <code>validateArg:83</code> or <code>validateVertexDescriptor:3700</code>. Still, the default reporter prints it out, so I printed it out the same way above.</p>
<p><code>type</code> I’m not 100% sure about, but, based on skimming what the assembly does with it, there seem to be three values for error (1) vs. warning (2) vs. “extended check” (??? because I haven’t been able to trigger it), and I think it only <code>abort</code>s on error, so I did the same thing.</p>
<p>Notice that if you return from this function instead of aborting, Metal just carries on as if you hadn’t failed a validation test.</p>
<h2 id="installing-the-block">Installing the block</h2>
<p>As mentioned earlier, the function is private (as are all of the Metal debugging functions). But, since it’s in the <code>Metal.framework</code>, which we link against, we can just <code>dlsym</code> it.</p>
<p>So, here’s a complete function that will install the failure block above, which you can call any time at startup (either before or after creating a device):</p>
<pre><code>BOOL installFailureBlock() {
    typedef void (^ReportFailureBlock)(int64_t type, const char *func, int32_t line, NSString *msg);
    typedef void MTLSetReportFailureBlock_t(ReportFailureBlock block);
    MTLSetReportFailureBlock_t *MTLSetReportFailureBlock = dlsym(RTLD_SELF, "MTLSetReportFailureBlock");
    if (!MTLSetReportFailureBlock) {
        NSLog(@"Oops, MTLSetReportFailureBlock seems to not exist on this platform!");
        return NO;
    }
    MTLSetReportFailureBlock(^(int64_t type, const char *func, int32_t line, NSString *msg) {
        NSLog(@"%s:%d: failed assertion `%@'", func, line, msg);
        if (type == 1) {
            abort();
        }
    });
    return YES;
}
</code></pre>
<p>If you want to store the function pointer in a global rather than a local, you can, but make it <code>static</code>, or give it a different name, or explicitly <code>dlopen</code> the <code>Metal.framework</code> instead of using <code>RTLD_SELF</code>; otherwise, <code>dlsym</code> will find your global pointer, and you’ll store a pointer to that pointer in your pointer, dawg.</p>
<h2 id="complete-api">Complete API</h2>
<p>It’s not hard to guess which functions are relevant to Metal debugging from the symbol dump of the Metal framework by their names.</p>
<p>Unfortunately, they’re all C functions, not C++ or ObjC, so you have to guess at the prototype or trial-and-error it. (First use Python <code>ctypes</code> to try calling stuff and see what doesn’t segfault, and then you can try stuff from inside a Metal app.)</p>
<p>Here are the bits relevant to report failure blocks:</p>
<ul>
<li><code>typedef void (^ReportFailureBlock)(int64_t type, const char *func, int32_t line, NSString *msg)</code></li>
<li><code>ReportFailureBlock MTLGetReportFailureBlock(void)</code></li>
<li><code>void MTLSetReportFailureBlock(ReportFailureBlock block)</code></li>
</ul>
<p>Notice that a library that needs to play nice would probably want to <code>MTLGetReportFailureBlock</code> and stash or chain up the result. But the default is null, and we can probably just assume nobody else in our process touches it.</p>
<p>Here’s the rest of the functions:</p>
<ul>
<li><code>int MTLValidationEnabled(void)</code>: Returns 1 when validation is enabled, and 0 when validation is disabled. And maybe 2 for extended, but I haven’t tested…</li>
<li><code>int MTLGetWarningMode(void)</code>: Returns 4 when validation is enabled. Not sure what that means, but it probably corresponds to one of the secret environment variables. And it seems to maybe be a bitfield, because if I set it to anything odd, I get type-2 failures for unused bindings.</li>
<li><code>void MTLSetWarningMode(int)</code>: You can change that 4 to something else, which could be handy if you knew what it meant.</li>
<li><code>int MTLFailureTypeGetEnabled(int type)</code>: Returns 1 on 0, 0 on 1-4, segfaults on anything else. What does that mean? I don’t know.</li>
<li><code>int MTLFailureTypeGetErrorModeType(int type)</code>: Returns 6 on 0, 4 on 1-4, segfaults on anything else. That 4 might be related to the warning mode in some way.</li>
<li><code>int MTLFailureTypeSetErrorModeType(int type, int errType)</code>: Change that 6 or 4 to something else, which presumably does something.</li>
<li><code>int MTLReportFailureTypeEnabled(int type)</code>: Returns 1 on 0, 0 on 1-4, segfaults on anything else.</li>
<li><code>void MTLReportFailure(???)</code>: This is the function called on validation errors and warnings. It takes a bunch of params, but I haven’t bothered to figure them all out.</li>
</ul>
<h2 id="portability">Portability</h2>
<p>As far as I can tell from the SDK dumps on <a href="https://ios.ddf.net">DDF</a> and generated headers on <a href="http://developer.limneos.net/index.php?ios=9.0&amp;framework=Metal.framework&amp;header=Metal-Symbols.h">limneos.net</a>, all of the Metal debugging functions have been there from the start of Metal on iOS. They’re also in at least the Mac 10.13-10.15 SDKs. And they work on at least Mac 10.14 and two versions of iOS. So, it should be safe to use them, at least until Mac 10.16 and iOS 14, and probably even longer.</p>
<h2 id="mtlreportfailure-details"><code>MTLReportFailure</code> details</h2>
<p>As mentioned above, every method that validates something calls <code>MTLReportFailure</code> if it fails. The arguments include the name and line, and a format string, and va_args to paste into that string, but there’s other things as well.</p>
<p>First it checks some stuff against <code>errorModes</code>  and <code>errorModes+23</code> and might skip over a bunch of code, but I’ve never traced through it doing this, so I don’t know. Probably this has to do with the modes and types stuff?</p>
<p>Then it formats up the message.</p>
<p>Then it checks <code>reportFailureBlock</code>, which is obviously the thing you set with <code>SetReportFailureBlock</code>. If you’ve set one, as covered above, it just calls that block and returns. If not, it carries on.</p>
<p>There are checks involving a bunch of things, including <a href="https://alastairs-place.net/blog/2013/01/10/interesting-os-x-crash-report-tidbits/"><code>gCRAnnotations.reserved3</code></a>, and <code>errorModes</code> again, and also <code>os_log_type_enabled(_os_log_default)</code>.</p>
<p>One thing this does is set up a string that will say <code>"error"</code> or <code>"warning"</code> or <code>"errorCheckExtended"</code> (I’ve only seen the first two happen), even though I don’t think that string ever gets used anywhere.</p>
<p>Another thing it does is possibly call <code>os_log_impl(_os_log_default)</code>. Which is exactly what we want. But I can’t tell how to trigger that. (Obviously it’s skipped if <code>os_log_type_enabled</code> is false, but even if it’s true, it gets skipped for other reasons.)</p>
<p>And ultimately, it calls <code>_assert_rtn</code>, the same function used by all SDK-internal asserts. Inside <code>_assert_rtn</code>, there are more checks against <code>gCRAnnotations</code>, but I don’t know how to get it to do anything other than what it normally does, which is to format a string to <code>stderr</code>, set up other reserved attributes in <code>gCRAnnotations</code> (which presumably affect the crash reporter), and <code>abort</code>.</p>
<p>At any rate, as much fun as it would be to debug the whole thing instead of just doing a quick skim, or to figure out how to inject a replacement in a way that works even on iOS, we’ve already got a way around it that’s less hacky, so I didn’t do either.</p>

