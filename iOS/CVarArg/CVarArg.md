# Variable Arguments in Swift

I was lately working on an iOS modernization project. We basically got an iOS app that is in production and, in fact, getting frequent updates.

The codebase was made in Objective-C, at a time that not even [ARC (Automatic Reference Counting)](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226) was around. Consequently, the app was plagued of memory-related bugs.

> Not just [memory leaks](https://en.wikipedia.org/wiki/Memory_leak), but also, [dangling pointers](https://en.wikipedia.org/wiki/Dangling_pointer).

As an initial measure we applied [Apple Instruments](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/) thoroughly, to detect and quickly fix those memory issues that were affecting the app the most.

But more as a long term solution, rather than converting the non-ARC Objective-C code to ARC Objective-C, we decided to slowly port the app to Swift 3 (which features ARC as a sole memory management technique).

> There were, definitely, a myriad of reasons that made us choose Swift instead of just flipping the existing non-ARC Objective-C into its ARC version.

In its third version, Swift is converging to a quite elegant syntax. Less verbosity, more conciseness than its Objective-C equivalent. A good thing, also, is that both Objective-C and Swift can easily interoperate at an arguably same [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)

> This last could be debatable: maybe the ABI isn't exactly the same but, from the perspective of an end-developer, you won't fall in compromises, being capable to expose your Objective-C types to your Swift ones and vice versa. The benefit of this, certainly, is the fact that your migration to Swift can be done in a paced fashion.

In spite of its already achieved maturity, you may eventually have to apply workarounds for a few thorns that you could get.

A one that we recently had to deal with, was the fact that C preprocessor macros aren't available in Swift. If we were writing a Swift app from scratch, we may never notice or find the need for preprocessor macros. But when you are porting Objective-C code to Swift that depends on such macros, then you realize that the feature is missing and you'll have to close the gap.

We didn't have too many macros, just one but quite useful:
```Objective-C
#ifdef DEBUG
#   define logd(fmt, ...) NSLog(fmt, ##__VA_ARGS__);
#else
#   define logd(...)
#endif
```
In a debug build, the macro resolves to an NSLog function call. In a release build, resolves to nothing instead. This latter point is relevant because it results in a smaller footprint as a consequence of this erasure.

Like I said, at the time of this writing this ability is missing in Swift. There are, though, ways to deal with this.

#### Approach 1: conditional compilation
This brute force technique consists in surrounding all debug traces with still-available `#if DEBUG` [conditional compilation statement](https://developer.apple.com/library/prerelease/content/documentation/Swift/Conceptual/Swift_Programming_Language/Statements.html#//apple_ref/doc/uid/TP40014097-CH33-ID538). The benefit of this is that it gets evaluated at compile time, so it will result in the erasure of all its content (i.e., everything inside the `#if DEBUG` and its corresponding `#endif`)

The problem of this is that it's not elegant. Developers must explicitly enter these conditional statements for each debug trace group.

> There's another drawback that irks me more: at the time of this writing at least, the Swift editor support in Xcode doesn't stick all conditional compilation statements to the left margin. Rather than that, it indents these like if they were regular code (they are not: conditional compilation statements get resolved, as the name suggests, at compile time).

#### Approach 2: encapsulated conditional compilation (proactive formatting)
A more concise way to write debug traces is by defining a logd function as follows:
```Swift
func logd(_ msg: String) {
#if DEBUG
    NSLog(msg)
#endif //DEBUG
}
```
This way, developers don't need to surround their debug traces into conditional compilation statements, because the definition of logd is already conditional: in a release build becomes an empty function that just discards its input.

It would work, but in non-debug builds it has two drawbacks:
* All calls to logd will get compiled and, therefore, will contribute to the resulting app footprint rather than getting erased.
* Even worse, the printable messages are resolved prior to the call to logd, leading to a daunting inefficiency.
> Typically, debug traces are used to expose run-time values like function results or intermediate values. Usually, these printable messages come formatted in such a way that developers get some context of what's being traced.
Formatting a string in Swift is as easy as:
```Swift
let pi = 3.14
print("Pi = \(pi)") // "Pi = 3.14"
```
The problem is not how to format a string in Swift, but the fact that this is an expensive operation. In particular, if for a debug trace, an unnecessary yet inefficient waste of resources that must be avoided.

It has to be a better approach.

#### Approach 3: encapsulated conditional compilation (lazy formatting)
This alternative leverages Swift's [_varargs_](https://developer.apple.com/reference/swift/cvararg) implementation. Like in the original NSLog logd macro, we provide a [string format specifier](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Strings/Articles/formatSpecifiers.html) and a list of arguments. This way, we postpone the formatting of the output until having confirmed that the debug trace must be indeed printed (i.e., if and only if we are effectively in debug mode).

The implementation of this stays as follows:
```Swift
func logd(_ format: String, _ args: CVarArg...) {
#if DEBUG
    NSLogv(format, getVaList(args))
#endif //DEBUG
}
```
Notice that I invoke the NSLogv function instead of NSLog. The reason is because I need to forward the list of arguments to NSLog in an unambiguous manner. If I called `NSLog(format, args)`, rather than passing a format and a variable list of arguments, I am passing a format and a single CVarArg argument (which at runtime looks like an array of `Any`, or an `[Any]`). NSLog will use this array as the format specifier first replaceable argument, and assume that all the rest are missing.

Rather than that, I use NSLogv, a variant of NSLog that takes a `CVaListPointer` (equivalent to C's [`va_list`](http://www.cprogramming.com/tutorial/c/lesson17.html) type).

> Needless to say that C got `va_list` to deal, precisely, with this [variable argument list forwarding issue](http://c-faq.com/varargs/handoff.html).

In release builds, this third approach can't avoid the invocation to logd with all the arguments placed in the runtime stack. However, in spite of this minor overhead, we skip all log formatting when running release builds.
