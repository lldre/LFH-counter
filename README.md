During Peter van Eeckhoutte's amazing `Corelan - Advanced` training, Peter mentioned that there were 1 or 2 papers about the LFH that confirmed the LFH gets activated for a particular size after 17 or 18 allocations. While this appeared to be accurate he seemed interested in understanding how this worked, so naturally:


![55e6024bc8ec05e5aac4d807c4040eec.png](./_resources/2ebcb43536f14350adbc3e6658e2ad6d.png)


> This was researched on a recent version of w10

# LFH activation
Before we look at the mechanics involved in triggering the LFH, let's quickly look at all the variables involved:
```c
ntdll!_HEAP
... snip ...
   +0x198 FrontEndHeap     : Ptr64 Void
... snip ...
   +0x1a2 FrontEndHeapType : UChar
... snip ...
   +0x1a8 FrontEndHeapUsageData : Ptr64 Wchar
   +0x1b0 FrontEndHeapMaximumIndex : Uint2B
   +0x1b2 FrontEndHeapStatusBitmap : [129] UChar
... snip ...
```

When an allocation is made `RtlpAllocateHeap()` decides on if it should use the lfh using the following steps:

* Starts with check if `(REQUESTED_ALLOCATION_SIZE + CHUNK_HEADER_SIZE) >> 4) + 1` (from here on reffered to as $index) is below `FrontEndHeapMaximumIndex` (initialised to 0x80, can also be 0x402). 
* Checks if the bit corresponding to the \$index is set in the `_HEAP->FrontEndHeapStatusBitmap` using the following calculation `FrontEndHeapStatusBitmap[$index >> 3] & (1 << ($index & 7))`. This bit, if set, indicates that the LFH is active for the bucket of that size.
* If the LFH is not yet active for that size it reads out `_HEAP->FrontEndHeapUsageData` and increments it by `0x21` (this is the counter mechanism).
* Right after incrementing it, it will bitwise AND the value to get the lowest 31 bits and check if the resulting value exceeds `0x10`
* It will then proceed to check the `_HEAP->FrontEndHeapType` for the value `2 (SEGMENT_HEAP)`. If that check is true it will load the pointer in `_HEAP->FrontEndHeap` (Which points to the LFH allocator perhaps) and pass it to `RtlpGetLFHContext`. If it's 0, the LFH isn't active for that heap and it will proceed to allocate it normally.
* If the LFH is available, it will set the corresponding bit inside the bitfield `_HEAP->FrontEndHeapStatusBitmap` that we calculated earlier, indicating an LFH has been created for that bucket.
* The last step is incrementing the `_HEAP->_HEAP_COUNTERS.AllocationIndicesActive` value.
* The next allocation of this size will be allocated in the LFH (allocation 18)

> It is good to know that the windows kernel ALSO implements this same mechanism to keep track of the number of allocations and LFH activations, albeit in a different type of structure. 

> Keep in mind when analysing that there are tools that disable the LFH for the default process heap, such as WinDbg (If the process is started under windbg rather than windbg attaching to a process).