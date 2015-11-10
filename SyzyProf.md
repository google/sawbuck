**DEPRECATED: Please use the [new version](https://code.google.com/p/syzygy/wiki/SyzyProf) of the page.**

# Introduction #

The end goal for SyzyProf is to be a turnkey instrumenting profiler that's able to instrument and profile any shipping or local build of Chrome on the spot, usable by anyone at any time, anywhere.

SyzyProf will (eventually) aggregate data across:
  * Multiple languages, notably C++, JavaScript (running in V8), and DART.
  * Multiple threads inside Chrome, across Tasks.
  * Multiple processes, across IPC boundaries.

SyzyProf generates output files that are compatible with [KCacheGrind](http://kcachegrind.sourceforge.net/html/Home.html), which is an excellent open-source profile visualization tool [that's available for Windows as a binary](http://sourceforge.net/projects/precompiledbin/files/kcachegrind.zip/download).

On Linux, kcachegrind might be as easy to come by as running this command:
```
sudo apt-get install kcachegrind
```

# Current Status #

SyzyProf is currently able to instrument and profile Chrome official builds or any local build linked with the /PROFILE flag.

SyzyProf captures invocation information per-thread, and can output per-thread aggregates as KCacheGrind "parts".

If you ask it nicely, SyzyProf will instrument your executable's import table (add `--instrument-imports` to the `instrument.exe` command line). In a per-thread output file, this allows you to easily answer questions such as which threads are calling ReadFile/WriteFile, which threads are creating windows, etc.

Integration has recently (as of July 2013) been added to V8 and to Chrome to allow profiling JavaScript in V8.
As of this writing, there isn't complete coverage for all of V8's internal functions which can result in dis-contiguous call graph for some JavaScript code.

# Details #

## How do I use it? ##

To use SyzyProf, you must:
  1. Download and install the latest SyzyProf installer from [the official build archive](http://syzygy-archive.commondatastorage.googleapis.com/index.html?path=builds/official/).
  1. Download and uncompress [the KCacheGrind zip archive](http://sourceforge.net/projects/precompiledbin/files/kcachegrind.zip/download). If you work at Google, you can instead download [this (newer) version of QCacheGrind, pre-built for Windows](https://goto.google.com/qcachegrind-win).
  1. Launch a SyzyProf shell from the SyzyProf folder in the start menu.
  1. Instrument your Chrome or Chromium build.
  1. Launch the call trace service.
  1. Run the instrumented Chrome instance.
  1. Aggregate the binary profile trace data with the grinder tool.
  1. Open the grinder output file in [KCacheGrind](http://kcachegrind.sourceforge.net/html/Home.html).
  1. Profit!

A profiling session might go like this:
```
> InstrumentChrome chrome-win32
> mkdir traces
> start call_trace_service.exe start --verbose --trace-dir=traces
> chrome-win32\chrome.exe --user-data-dir=c:\temp\killmenow
    ... time passes ...
> call_trace_service.exe stop
> grinder.exe --mode=profile --output-file=chrome.callgrind traces\*.bin 
> kcachegrind.exe chrome.callgrind
```

## Screenshot ##

This will open the callgrind file in KCacheGrind (or QCacheGrind) which will look something like this:

![http://sawbuck.googlecode.com/files/chrome-calltrace.png](http://sawbuck.googlecode.com/files/chrome-calltrace.png)

Some things to note about this screenshot are:
  * The call graph is presented visually, with the costliest calls highlighted.
  * The SyzyProf Grinder outputs a profile part per thread. This means that for any given function, you can view its cost in an individual thread, across a selection of multiple threads, or across all threads in a process.
  * If you're profiling a locally built version of Chrome, you'll be able to view the sources [annotated with profiling data](http://kcachegrind.sourceforge.net/html/Shot4.html).
  * QCacheGrind **rocks**.

## Caveats ##

Note that SyzyProf needs the binary to be linked with the "/PROFILE" flag set, and that SyzyProf cannot at the present time instrument official Chrome binaries for boring reasons that will eventually be remedied.

## How does it work? ##

SyzyProf uses binary instrumentation to get a hook on every function entry, and then employs "return address swizzling" to get an exit hook. This is done by grabbing the return address on the stack at function entry, pushing it to a "shadow stack", and replacing it with the address of the exit hook.

Each exit hook is an executable thunk, allocated from a thread-local stash. Each thunk contains a short instruction sequence that calls the main exit hook, and immediately following the thunk is the shadow stack data for that level of the shadow stack. This is more robust and performant than an explicit shadow stack, especially in the presence of non-local gotos and exceptions (which can implicitly pop the machine stack), as well as assembly tricks that can arbitrarily mess with the stack.

The swizzling creates a problem with V8, which looks at the return addresses of certain runtime functions to find the JIT/JS code that invoked them. To work around this, V8 has been enhanced with a profiling hook, which gives the profiler an opportunity to resolve an address on stack, to the location of the profiler's stash of the original return address. Search for "SetReturnAddressLocationResolver" in [the V8 API include file](http://code.google.com/p/v8/source/browse/trunk/include/v8.h). Chrome has then been augmented to set this hook when running under instrumentation.