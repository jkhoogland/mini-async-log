
Useful Features Only (UFO) Logger
-----------
A feature reduced asynchronous data logger. Sponsored by my employer **Diadrom AB.**

We just wanted an asynchronous logger that can be used from many dynamically loaded libraries without doing link-time hacks like linking static and hiding symbols and some other features.

After having maintained a slightly modified version of glog and given the fact that this is a very small project we decided that existing wheels weren't round enough.


## Design rationale ##

 - Simple. 
 - Not over abstracted and feature packed, easy to figure out what the code is doing, easy to modify.
 - Low latency, fast for the caller.
 - Asynchronous (synchronous calls can be made for special messages, but they have grotesque overhead).
 - No string formatting in the calling thread, the data is raw copied. 
 - One conditional call overhead for inactive severities.
 - No ostreams (a very ugly part of C++ for my liking), just format strings checked at compile time (if the compiler supports it) with type safe values.
 - No singleton by design, usable from dynamically loaded libraries. You provide the instance either explicitly or by using Koenig lookup.
 - Suitable for soft-realtime work. Once it's initialized the fast-path can be clear from heap allocations if you configure it properly.
 - File rotation-slicing
 - Targeting g++4.7 and VS 2010
 - Boost dependencies just for parts that will eventually go to the C++ standard.

## How does it work ##

It just borrows ideas from many of the loggers out there.

When the user is to write a log message, the size is precomputed, then the memory is reclaimed either from the fixed size free-list or the heap (configurable) and then message is serialized and passed to the worker thread queue as an intrusive linked list node.

The user thread doesn't format strings, just copies built-in type values and whole program duration C string pointers (Deep copies can be done if required too) to the message and appends data for the worker thread to be able to decode it. This is restrictive but gives other benefits too.

The messages are formatted by using printf-style strings, where the formatting string is required to be a literal (not a const char*), e.g:

log_error ("the value of i is {} and the value of j is  {}", i, j);

The function is type-safe, when "constexpr" and variadic template parameters are available it is matched with the parameters at compile time, otherwise the errors are caught at run time in the log file.

> see this [example](https://github.com/RafaGago/ufo-log/blob/master/example/overview.cpp)

I might work in applying a little "compression" to the integer types like protobuf does (but simpler) to try to pack the messages more in case no use of the heap is allowed.

## Benchmarks ##
I used to have some benchmarks here showing that "my new thing (TM)" is the best invention since sliced-bread, but as the benchmarks were mostly an apples-to-oranges comparison (e.g. comparing with glog which is half-synchronous) I just removed them.

With the actual performance and for simple messages it takes the same time to retrieve the time stamp at the producer side with the chrono library than to build and enqueue the message. So I won't bother optimizing.

## File rotation ##

The library can rotate fixed size log files.

Using the current C++11 standard files can just be created, modified and deleted. There is no way to list a directory, so the user is required to pass at start time the list of files generated by previous runs. I may add support for boost::filesystem /std::filesystem, but just as an optional but ready external code, so everyone can skip this heavy dependency. There is an example using boost::filesystem in the "/extras" folder

## Initialization ##

The library isn't a singleton, so the user should provide the front-end instance.

There are two methods, one is to provide it explicitly and the other one is by accessing a global function.

If no instance is provided, the global function "get_ufo_logger_instance()" will be called without being namespace qualified, so you can use Koenig lookup/ADL.

The name of the function can be changed at compile time, by defining UFO_GET_LOGGER_INSTANCE_FUNCNAME.

Be aware that it's dangerous to have a dynamic library or executable loaded multiple times logging to the same folder and rotating files each other. Workarounds exists, you can prepend the folder name with the process name and ID, disable rotation and manage rotation externally (e.g. by using logrotate), etc.

## Termination ##

The worker blocks in its destructor until its work queue is empty when normally exiting a program.

When a signal is sent you can call the frontend function  [on termination](https://github.com/RafaGago/ufo-log/blob/master/include/ufo_log/frontend.hpp) after shutting down your data/log producers. This will early interrupt any synchronous calls you made.


## Errors ##

As for now, every function returns a boolean if it succeeded or false if it didn't. A filtered out/below severity call returns true.

The only two possible failures are either to be unable to allocate memory for a log entry or an asynchronous call that was interrupted by "on_termination".

The functions never throw.

## Weaknesses ##

 1. No C++ ostream support. (not sure if it's a good or a bad thing...)
 2. Limited formatting abilities (it can be improved with more parser complexity).
 3. No way to output runtime strings/ memory regions without deep-copying them.
 
The third point is the most restrictive for my liking, it's just inherent to the asynchronous/non-blocking design, there is no guarantee about the passed data lifetime.

It's possible to artificially increment the refcount of a shared_ptr by copying it to an instance created using "placement_new" and to decrement it in the worker using the same trick, I keep this idea on hold for now.

## Using the library ##

You can compile the files in the "src" folder and make a .dll/.so file or just use the whole thing compiled in your project.

If you want to compile the library inside your project you need to merge the "src" and "include" folders (or to add both as an include directory to the compiler) and to compile "frontend_def.hpp" in one translation unit.

## Todo ##

 - Test in Windows.
 - makefile
 - Visual Studio project.

## Disclaimer ##

THIS PROJECT IS UNDER DEVELOPMENT AND SHOULDN'T BE CONSIDERED STABLE BY NOW. Even tough it seems to work.

> Written with [StackEdit](https://stackedit.io/).




