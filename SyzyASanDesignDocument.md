**DEPRECATED: Please use the [new version](https://code.google.com/p/syzygy/wiki/SyzyASanDesignDocument) of the page.**

#summary An address sanitizer based on Syzygy.



# Introduction #

The goal of SyzyASan is to give to the Windows developers of Chrome a tool to easily detect end fix the memory errors. We want to reduce as much as possible the performance’s impact to be able to run SyzyASan in a bot on the Chrome’s memory waterfall to ensure that a patch don’t introduce a memory error (directly or not).

# Background #

The Syzygy project provides an API to decompose a PE binary into a graph of basic-blocks, it give us the ability to walk through the instructions of those blocks and inject additional code into them. With that information we should be able instrument code in order to detect memory access errors in Chrome DLL.

The [Address Sanitizer](https://code.google.com/p/address-sanitizer/) team is currently working on a similar project. The main differences between that project and SyzyASan are:
  * Their project is based on Clang.
  * Currently, it supports only x86 Linux and Mac; SyzyASan is Windows specific.
  * Their instrumentation is made at compile time. It is unable to catch memory errors in statically linked libraries. SyzyASan instruments the binary post-link, including any statically linked libraries.

# Overview #

There are several different kinds of memory error:
  * **OOB** - Out-of-bounds accesses (buffer overflow/underflow)
    * Heap
    * Stack
    * Globals
  * **UAF** - Use-after-free (dangling pointer)
  * **UAR** - Use-after-return
  * **UMR** - Uninitialized memory reads
  * Leaks
  * Double/Invalid free
  * Overapping memcpy parameters
  * **…**

Because SyzyASAN works directly on the binary (without looking at the source code), the kind of errors it can find are limited. Currently its scope is limited to checking heap access. That means that it finds the places where a function tries to access some invalid addresses (OOB, UAF principally).

The features it needs to support this are
  * A heap red/green-zoning instrumentation tool;
  * The ability to detect the place where the heap is accessed and to log an error if the access is to an invalid location. It may be desirable to limit this to writes accesses;
  * A way to measure the impact of the instrumentation on the performance of the application.
  * A clear way to detect the origin of the error once a bad memory access is detected.

# Detailed Design #

Here is the high level algorithm:
  * Initialize a memory chunk to store the shadow memory metadata.
  * Replace the heap-related kernel32 calls with a run-time library that maintains the shadow memory.
  * Find every instruction that writes/read memory.
  * For each of them inject some code to:
    * Calculate the shadow memory address for this address.
    * Check if this address is poisoned, if so throw an error, otherwise continue.

We can distinguish several parts in this approach, each of them will be described in the following.

## Allocation and initialisation of the shadow memory ##

The shadow memory is managed by an all-static class called Shadow. Knowing that the allocator in a 32-bit application will always return an 8-byte aligned address, we can map 8 bytes of the application memory into 1 byte of the shadow memory. Thus, the size of the shadow memory is equal to 232-3 bytes. To map an address of the application memory to its shadow memory address we just have to divide the address by 8 (shift it 3 bits to the right).

The algorithm to poison an address is the following:
  * Shift the address and the size of the allocation 3 bits to the right. It will give us the index where the red-zone should start in the shadow memory.
  * If the 3 LSBs of the address were not equal to zero, save their value into shadow\_memory(index) and increase the index.
  * Set the byte from shadow\_memory(index) to shadow\_memory(index + size) to 0xFF (or another value greater than 7) to mark them as red-zoned.

The algorithm to un-poison an address is simply the opposite of this one. We store the value 0 in the shadow memory bytes of a non-poisoned memory address.

Checking if a memory address is poisoned is simple:
  * Shift the address 3 bits to the right. It give us the index of this address in the shadow memory.
  * If shadow\_memory(index) is 0, the access is safe. If it’s the red-zone value (0xFF in our case) the the access is not safe.
  * If shadow\_memory(index) is > 0 but < 8 the access is safe if the value stored in the 3 LSBs of the address checked is inferior to shadow\_memory(index), otherwise it’s unsafe.

## Hooking of the allocators ##

Currently we redirect some heap related functions from kernel32 to our run-time (HeapCreate, HeapSetInformation, HeapAlloc, HeapFree, HeapReAlloc,HeapSize,Heap...). Our function are contained in a class named HeapProxy, who act as a proxy between the application and kernel32.dll. I.e, we redirect HeapAlloc to asan\_HeapAlloc, who call HeapProxy::HeapAlloc who initialize the shadow memory corresponding to this allocation and call kernel32::HeapAlloc to allocate the memory.

For each memory allocation we add a red-zoned header and a tail, this allow us to easily detect an over/under-flow access to this memory area. We also use the red-zoned memory of the header to store some information about this memory block: size, state, stack pointers on allocation, stacks pointers on free... Those information allow us to give to the user more information when a bad access is detected.

## Instrumentation of the code ##

We’re using the Syzygy’s API to instrument the code. With the basic-block decomposition we’ve been able to set-up a transformation that check every instruction of a basic block, and each time that we find a memory access we instrument it like this:
```
mov eax, [address]
```
become:
```
push edx
lea edx, [address]
call ensure_address_is_not_poisoned
mov eax, [address]
```