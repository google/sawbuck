**DEPRECATED: Please use the [new version](https://code.google.com/p/syzygy/wiki/SyzygyDesign) of the page.**

#summary Design overview for the Syzygy toolchain.

# Syzygy - profile guided, post-link executable reordering. #

## Contents ##



## Objective ##

Syzygy is a suite of tools to perform profile-guided, function-level reordering of 32-bit Windows PE executables, to optimize their layout for improved performance, notably for improved paging patterns.

Syzygy generates accurate symbol information files for the reordered binaries, to make it possible to debug reordered binaries, and to analyze crash reports involving reordered binaries.

The main initial goal is to accelerate Google Chrome cold start by reducing the volume of paging traffic by better than 80%, and by shaping the remaining paging traffic so that the executable file is read linearly from disk.
As a secondary goal, Syzygy aims to reduce the working set contribution from image code and data by 40% or so.

## Background ##

The default symbol ordering chosen by the Microsoft Visual Studio linker does not take execution order into account, so the symbol ordering is effectively random with respect to the execution order. This causes a lot of unnecessary and awkwardly shaped paging traffic on cold start, and results in poor locality of code and data within each page in the image. The poor locality, in turn, results in a working set that's larger than necessary and an elevated rate of minor page faults.

Aside from improving on cold start times and working set density, improving the executable layout can also improve cache efficiencies for the [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer), instruction and data caches.

Developers on both the Open Office and the Firefox projects have noticed lately that paging traffic is a large contributor to cold start launch times, and have spent some time trying to analyze and optimize this.

See e.g. these two pages:
  * [Open Office - reorder symbols for libraries](http://wiki.services.openoffice.org/wiki/Performance/Reorder_Symbols_For_Libraries).
  * [Firefox - Windows sucks at memory-mapped IO](http://blog.mozilla.com/tglek/2010/04/19/windows-sucks-at-memory-mapped-io-during-startup/).

Both Firefox and Open Office have been down essentially the same path, they've:
  1. found paging traffic to be a large contributor to cold start launch times.
  1. used existing tools, such as [Xperf](http://msdn.microsoft.com/en-us/performance/cc825801.aspx), to look at the shape and volume of the paging traffic.
  1. used the Visual C++ compiler's profile hook to gather information about execution order.
  1. used the Visual Studio linker to reorder the binaries by passing it an ordering file to the **/order** flag.

Both projects have abandoned this as they haven't been able to demonstrate an appreciable improvement in paging traffic or cold start times through this approach. See the [caveats section](#Caveats.md) for a discussion of why this approach is non-viable. However, the Firefox project has made [some strides on Linux](http://blog.mozilla.com/tglek/2010/04/12/squeezing-every-last-bit-of-performance-out-of-the-linux-toolchain/), where [Taras Glek](http://blog.mozilla.com/tglek/) wrote [Icegrind](http://blog.mozilla.com/tglek/2010/04/07/icegrind-valgrind-plugin-for-optimizing-cold-startup/), a plugin to Valgrind that records information about the code and data accessed during Firefox startup. Taras has been able to show a 28% startup time improvement for Firefox on Linux, which demonstrates what can be accomplished with accurate data and full control of ordering.

Microsoft has been reordering the Windows system and application binaries to optimize them for paging reduction, using a toolset reported to be derived from and/or built on top of the [Vulcan research project](http://research.microsoft.com/pubs/69850/tr-2001-50.pdf). Relatively little else is known about the tools or the methods they use. Looking at Windows system binaries like **ntoskrnl.exe**, it's apparent that their layout doesn't correspond with the Visual Studio toolchain's defaults, and their respective symbol files contain [OMAP re-mapping information](http://msdn.microsoft.com/en-us/library/bb870605(VS.85).aspx). The Microsoft [Binary Technologies Projects](http://www.microsoft.com/windows/cse/bit_projects.mspx) may be involved in this.

## Overview ##

The Syzygy suite of tools performs profile-guided, post-link reordering of Windows PE binaries.
The tool suite is composed of the following components:
  1. The **instrumenter** tool reads a PE image file and its symbol information, and writes an instrumented version of the image file to disk.
  1. The **calltrace profiler** is a DLL exporting a profile hook that gets invoked for every function entry in the instrumented image file. The calltrace profiler yields a binary log file containing the addresses of the functions invoked along with timing information and process/thread affinity.
  1. The **ordering generator** reads the log file, then applies heuristics and an ordering strategy to come up with an improved ordering for the original binary.
  1. The **relinker** tool reads the original image file, its symbols and the ordering file, and outputs a reordered image file and a corresponding symbol file.

By using binary instrumentation, Syzygy achieves full coverage for all functions in the executable, even for thunks created by the compiler, functions linked from libraries or import thunks.

By capturing and processing a full trace of the functions invoked and their process/thread affinity, the **ordering generator** can discover not only temporal function relationships, but also process or thread specific clusterings. For Chrome in particular, it stands to reason that the browser process will behave very differently than a renderer process which will again behave differently than a plugin host process.

By doing post-link reordering of the binaries, Syzygy gains full control over the order of the final image file, which allows the toolchain to attain the best possible results. This comes at the cost of some implementation complexity; see the detailed design for details.

### Current Results ###

The first results, gathered towards the end of Q1 2011, are very promising.
By ordering chrome.dll code in strict order of first appearance, the number of hard page faults **from code** in chrome.dll in the Chrome browser process on cold start was reduced from 323 to 61, a reduction of 81%.

These results were achieved by instrumenting Chrome release version 10.0.648.133, profiling the instrumented binary on a fresh install of Windows XP Professional with SP3, and ordering the chrome.dll binary according to that profile. The hard page fault measurements were done by running the Chrome process from a newly mounted shadow copy of the C-drive file system, using the volume shadow copy service, while capturing an NT Kernel trace. In an attempt to minimize caching effects, each of the tests were run 6-7 times before capturing the data illustrated here. In each case, the run consisted of launching Chrome.exe to the new tab page using an existing, non-default profile on the C drive.

Since this was captured in a virtual machine, wall clock times should still be viewed skeptically.

Here are two graphs illustrating the page fault patterns in chrome.dll before and after reordering, respectively. The red points are hard page faults whereas the green points are soft faults, and the yellow line in the graph is the launch time of the first (and only) Chrome renderer process.

### Page faults before reordering. ###

<img src='http://sawbuck.googlecode.com/files/graph-before.png' width='100%' />

### Page faults after reordering. ###

<img src='http://sawbuck.googlecode.com/files/graph-after.png' width='100%' />

In the version of chrome.dll under inspection here, code occupies relative addresses 0x00000000 through 0x00E66200; notice how the page fault pattern in that range of addresses is a pleasing, monotonically increasing pattern. The upper half of the latter image still contains the same random pattern, as data has not been reordered.

Notice also that the launch of the second renderer process occurs at around 2.25 seconds in the reordered case, as opposed to at around 3.95 seconds in the unordered case.

## Detailed Design ##

### Usage ###

The Syzygy toolchain consists of:
  1. An instrumenter tool: a command line executable that injects instrumentation code into the original binary.
  1. A calltrace generator: a DLL the instrumented binary depends on.
  1. An order generator: a Python script that reads the calltrace logs and generates an order file.
  1. A relinker: a command line executable that reads the original executable, its symbol file and the order file, and outputs a relinked binary and the associated symbol file.

To use the toolset to reorder a binary the user needs to:
  1. Instrument the binary to reorder.
  1. Initiate Event Tracing for Windows (ETW) log capture, either programmatically or by invoking the appropriate **logman.exe** command line invocations.
  1. Run the instrumented binary through the activity or activities it is to be optimized for.
  1. Stop the ETW log capture.
  1. Process the logs with the **order generator**.
  1. Feed the resultant order file, the original executable and its symbol file through the **relinker**.

This will result in a reordered executable and a new symbol file for the new executable.

### Implementation ###

In order to achieve the profiling and post-link reordering of Windows PE binaries, Syzygy needs to be able to:
  1. Instrument the original PE image file by injecting an invocation of a profile hook prior to any function entry.
  1. Decompose the original PE image file into a set of code and data blocks and their references.
  1. Process the captured call trace and produce an ordering.
  1. Recompose the code and data blocks into another, functionally equivalent PE image file.
  1. Update the original symbol file with information about the new ordering.

The rest of this section is a discussion of these in turn.

#### Instrumentation ####

In order to capture accurate data about in which order functions are invoked, the **instrumenter** injects instrumentation code into the binary.
This is done by:
  * Injecting a new [import table entry](http://en.wikipedia.org/wiki/Portable_executable#Import_Table) for the **calltrace profiler** into the binary.
  * Locating every function block in the original binary.
  * For each function "foo", create a profiling thunk "foo\_thunk".
  * Append all the "foo\_thunk" thunks to the binary.
  * Locate every reference to "foo", and redirect it to "foo\_thunk".

The "foo\_thunk" profiling thunks look like this:

```
  push foo
  jmp [__imp_indirect_penter]
```

i.e. the profiling thunk invokes the _indirect\_penter function, which is exported from the **calltrace profiler** DLL with the original function as a parameter on the stack. The instrumentation thunks are appended to the image by appending a new code segment named ".thunks" to it. Note that since the **foo** and **imp\_indirect\_penter** references are absolute, this also necessitates adding to and rewriting the relocations section._

This method has proven easy to implement, and quite comprehensive compared to other, perhaps more conventional methods. It also has the advantage that the original image's functions and data are not moved around, and so the original symbol information is sufficient for debugging the instrumented binary and for resolving the addresses in the captured call trace back to human-readable symbols.

Preamble patching, as an alternative case in point, involves overwriting the start of the function with a **jmp** instruction, and relocating the instructions overwritten to somewhere else. This does not let us profile functions that are shorter than 5 bytes long, nor functions that contain a jump or a branch to the first five bytes of their body.

#### Decomposition ####

In order to safely move code and data, Syzygy must first decompose the PE Image file into a graph of code and data blocks, and locate all references across and within those blocks. After the blocks have been moved around, the references must be rewritten to their new destinations. Breaking the PE file into blocks is done by referring to the symbols for the executable, and is a relatively simple process. Finding all references is however a more involved process, which involves referring to data in the image file itself.

In a Windows x86 PE image file, the references are of four kinds:
  1. Absolute references.
  1. PC-relative references.
  1. Relative virtual addresses (RVA) are relative to the start of the image.
  1. File offset addresses are addressed by file index to a structure in the PE image.

The absolute references are found by walking the PE Image file's relocation entries. Each relocation entry calls out a location in the binary that contains an absolute pointer. This list of relocation entries must be complete in order to allow the Windows loader to relocate the image file, e.g. in the case where its preferred load address is occupied or for address space randomization.

The PC-relative references are located by disassembling the functions in the binary. It is to be noted that the disassembly must achieve complete coverage, or there's a risk that some PC-relative references are not found and would not be rewritten on relocating the function.

Getting full disassembly coverage is slightly more difficult than it appears at first blush because of computed jumps. Computed jumps appear primarily in two cases:
  1. In tail-call elimination where the function called is a virtual member function.
  1. In the code generated for large switch statements.

Both cases can be covered by heuristics or by implementing full data flow analysis over the disassembled code. As it turns out, there is a shortcut to this because the Visual Studio tool chain leaves a trail of breadcrumbs to make this possible. Whenever a function is generated that has non-contiguous control flow, the tool chain emits a label for each location in the function that's the destination of a computed jump. This makes sense since other tools in the Visual Studio tool chain, such as the code coverage profiler, need this information to implement their instrumentation and profiling.

##### In memory representation #####

The in-memory representation of the binary PE image decomposition is an instance of the [BlockGraph class](http://code.google.com/p/sawbuck/source/browse/trunk/syzygy/core/block_graph.h). The image contains a set of blocks where each block points to its object code in the original image and has a set of references. Each such reference has a source block offset, a size, a type (Absolute, PC-relative, etc), a destination block and index.

This is a decent level of abstraction for the purpose, and has the advantage that it's architecture independent, so the same level of abstraction could be used to manipulate binaries on other architectures e.g. ARM.

### Ordering Generation ###

The **calltrace profiler** writes the calltrace event stream to a binary log file using [Event Tracing for Windows](http://msdn.microsoft.com/en-us/library/bb968803(VS.85).aspx), where each log record contains a batch of {function address, time of entry} events. This log, combined with events from an [NT Kernel Logger session](http://msdn.microsoft.com/en-us/library/aa363691(VS.85).aspx) which provides information about processes, threads, and module load/unload events, is processed by the **order generator**. The two log streams combined contain sufficient information to resolve absolute addresses in the log to a function address within a module. Through the process/thread affinity, it's also possible to infer whether a function entry occurred in a Chrome renderer or browser process if so desired.

The **order generator** distills this information, applies an ordering strategy, and writes a file that specifies the ordering to apply.
The current ordering used is "first order of appearance", which is optimal for cold start paging patterns, but this will be refined over time.

The **order generator** has complete and accurate information about function entries. This enables it to compute an optimal ordering for the functions in an executable given a particular calltrace. It does not, however, have information about which execution paths were exercised within the function, which means it can not know precisely which data the function may have read or written. However, using the pessimistic estimate of assuming the function touched all data it refers as a basis for ordering data in the executable is likely to improve data locality.

### Recomposition ###

Recomposition of the reordered PE image is the process of:
  * iterating over the blocks in the reordered abstract image,
  * updating the values of references within, and
  * writing them to their correct location in the image file, before
  * writing a new relocation section and updating the PE image headers.

### Symbol Updating ###

The symbol updating involves adding two address relocation streams to the PDB file. The two streams are the **OmapTo** and the **OmapFrom** streams which contain forward and backward mappings between addresses in the original and reordered binary.

This is primarily tedious because the PDB file format is undocumented. However, the [Common Compiler Infrastructure](http://ccimetadata.codeplex.com/) project, a Microsoft open source project, has shed light on the PDB file format and the structures involved in describing Omap data. From examination of some of that code, it appears that adding the Omap data to the PDB file involves adding the two new streams to the Pdb file and updating the DbiDbg header in the Dbi info stream.

The PDB file format is much like compound documents in that each PDB file contains multiple streams of distinct information. The streams are however not hierarchical, nor named, as in compound documents, but rather are referred by their ordinal number. The OmapTo and OmapFrom address maps are stored in two distinct streams, and their respective stream numbers are stored in a DbiDbg Header. This structure seems to appear at the end of the Dbi information stream, which itself has a fixed stream number, and is a concatenation of parts described by fields in the Dbi header.

Here are links to the description of the Dbi header and the optional dbg header:
  1. [Dbi Header](http://ccimetadata.codeplex.com/SourceControl/changeset/view/52123#96530)
  1. [DbiDbg Header](http://ccimetadata.codeplex.com/SourceControl/changeset/view/52123#96529)

## Caveats ##

The Visual C++ toolchain is used in a mode called "link time code generation" when producing the production Chrome binaries. In this mode, the C++ compiler produces object files that are not in standard COFF format, and contain some form of intermediate code. The actual code generation is deferred until link time, at which point the linker has access to the intermediate code to the whole program. This allows the linker to do cross compilation-unit optimization such as inlining, custom calling conventions, etc. The downside to this for Syzygy is that the toolchain is opaque from the point where source code is consumed by the compiler to the point where a binary has been produced by the linker.

### Using the compiler's profile hook. ###

After attempting to use the Visual C++ profile hook and analyzing the results, it became apparent that this approach is non-viable for several reasons:
  * The Visual Studio link time code generation is perturbed by the presence of the profiling code. Building the 437 release branch of Chrome with and without profiling, then comparing the symbols in the two binaries showed that the profiled binary contained approximately 5000 extra symbols. There were also some 1943 symbols in the original binary that didn't appear in the profiled one.
  * The profiling hook is only inserted in code compiled with the requisite compiler flag. This excludes code linked from libraries such as the Microsoft-supplied CRT, import thunks etc.
  * The Visual Studio linker has severe limitations on which symbols it will order through the [/ORDER](http://msdn.microsoft.com/en-us/library/00kh39zz(VS.71).aspx) flag. The linker will only order "public" symbols, which means that static functions, compiler generated initializers and import thunks cannot be ordered through this method. Additionally, only symbols in object files compiled with the [\*/Gy\* Enable Function-Level Linking"](http://msdn.microsoft.com/en-us/library/xsa71f43(VS.80).aspx) flag can be ordered. This excludes the Microsoft-supplied CRT. Another limitation is that linker simply crashes if the name of the function in the supplied order file is very long (> 2500 chars). STL is known to produce very long function names.

Between the measurement error and the ordering restrictions, it's not surprising that the results are lackluster.

To better illustrate why this is, note that each stray symbol in the code segment will result in loading the containing 32Kb chunk, so 32 stray symbols can cause the faulting of a megabyte of code.

### Ordering and optimizing at the basic block level ###

Since Syzygy must accurately disassemble the entire binary, it could compute the inner structure of functions down to the basic block level. From there, it's not a long stretch to profile, order and perhaps even optimize down to the basic block level. Easy and potentially gainful optimizations at that level would be to move infrequently used code paths out of the way for better instruction cache utilization, and to optimize function layout for branch direction.

This obviously involves additional complexity, and so is out of scope for the initial implementation.