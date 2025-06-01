.. warning::

   This document is currently under construction and may be incomplete or subject to significant changes.
   Please check back later for updates, and consult the instructor if you are unsure about any missing parts.


================
Lab 3: Allocator
================

############
Introduction
############

A kernel allocates physical memory for maintaining its internal states and user programs' use.
Without memory allocators, you need to statically partition the physical memory into several memory pools for
different objects.
It's sufficient for some systems that run known applications on known devices.
Yet, general-purpose operating systems that run diverse applications on diverse devices determine the use and amount
of physical memory at runtime. On VF2, available physical memory and reserved regions are described by the Device Tree,
so you must read those descriptions at boot.

In Lab 3, you need to implement memory allocators that rely on VF2 Device Tree for memory layout.
They’ll be used in all later labs.


#################
Goals of this lab
#################

* Implement a Page Frame Allocator that reads memory ranges from the VF2 Device Tree.

* Implement a Dynamic Memory Allocator that obtains pages from the VF2 Page Frame Allocator.

* Implement a Startup Allocator based on a bump pointer, using VF2 Device Tree <memory> and <reserved-memory> entries.


##########
Background
##########

Reserved Memory
================

After VF2 is booted, all reserved memory regions—including spin tables (if required by the platform),
the device tree blob, the kernel image, initramfs, and any platform-specific reserved areas—
must be provided by the VF2 Device Tree <reserved-memory> nodes.
Your Startup Allocator should read these <reserved-memory> entries at startup,
mark the corresponding physical frames as reserved, and not allocate them.

Dynamic Memory Allocator
========================

Given the allocation size,
a dynamic memory allocator needs to find a large enough contiguous memory block and return the pointer to it.
Also, the memory block would be released after use.
A dynamic memory allocator should be able to reuse the memory block for another memory allocation.
Reserved or allocated memory should not be allocated again to prevent data corruption.

Page Frame Allocator
======================

You may be wondering why you are asked to implement another memory allocator for page frames.
Isn't a well-designed dynamic memory allocator enough for all dynamic memory allocation cases?
Indeed, you don't need it if you run applications in kernel space only.

However, if you need to run user space applications with virtual memory,
you'll need a lot of 4KB memory blocks with 4KB memory alignment called page frames.
That's because 4KB is the unit of virtual memory mapping.
A regular dynamic memory allocator that uses a size header before memory block turns out half of the physical memory
can't be used as page frames.
Therefore, a better approach is representing available physical memory as page frames.
The page frame allocator reserves and uses an additional page frame array to bookkeep the use of page frames.
The page frame allocator should be able to allocate contiguous page frames for allocating large buffers.
For fine-grained memory allocation, a dynamic memory allocator can allocate a page frame first then cut it into chunks.

Observability of Allocators
============================

It's hard to observe the internal state of a memory allocator and hence hard to demo.
To check the correctness of your allocator on VF2, you need to **print the log of each allocation and free**
over the VF2 UART driver (initialized via the Device Tree’s chosen stdout-path).

.. note::
  TAs will verify correctness by these logs during the demo. Prefix each log message with “[VF2 Allocator] ”
  to differentiate from other kernel output.


###############
Basic Exercises
###############

In the basic part, your allocator can focus on a single allocable memory region provided by the VF2 Device Tree’s <memory> node
and does not need to handle reserved-memory entries. Choose one available memory segment from the Device Tree
(for example, the first <memory> region) and manage that part of memory only.

Basic Exercise 1 - Buddy System - 40%
=====================================

Buddy system is a well-known and simple algorithm for allocating contiguous memory blocks.
It has an internal fragmentation problem, but it's still suitable for page frame allocation 
because the problem can be reduced with the dynamic memory allocator.
We provide one possible implementation in the following part.
You can still design it yourself as long as you follow the specification of the buddy system.

.. admonition:: Todo

    Implement the buddy system for contiguous page frames allocation on VF2. Your maximum order must be larger than 5.
    Ensure that you compute page frame counts and addresses using the <memory> regions read from the VF2 Device Tree,
    skipping any holes.

.. note::

  You don't need to handle the case of out-of-memory.

Data Structure
----------------

**The Frame Array** (or *"The Array"*, so to speak)

*The Array* represents the allocation status of the memory by constructing a 1-1 relationship between the physical memory frame and *The Array*'s entries.
For example, if VF2 Device Tree indicates two allocable regions totaling 200 KiB with each frame being 4 KiB,
then The Array would consist of 50 entries. Use the base addresses from the <memory> regions to compute
each frame’s physical address (e.g., if the first region begins at 0x8000_0000, its first entry represents 0x8000_0000,
next 0x8000_1000, etc.).

However, to describe a living Buddy system with *The Array*, we need to provide extra meaning to items in *The Array* by assigning values to them, defined as followed:

For each entry in *The Array* with index :math:`\text{idx}` and value :math:`\text{val}`
  (Suppose the framesize to be ``4kb``)

  if :math:`\text{val} \geq 0`:
    There is an allocable, contiguous memory that starts from the :math:`\text{idx}`'th frame with :math:`\text{size} = 2^{\text{val}}` :math:`\times` ``4kb``.

  if :math:`\text{val} = \text{<F>}`: (user defined value)
    The :math:`\text{idx}`'th frame is free, but it belongs to a larger contiguous memory block. Hence, buddy system doesn't directly allocate it.

  if :math:`\text{val} = \text{<X>}`: (user defined value)
    The :math:`\text{idx}`'th frame is already allocated, hence not allocable.

.. image:: /images/buddy_frame_array.svg

Below is the generalized view of **The Frame Array**:

.. image:: /images/buddy.svg


You can calculate the address and the size of the contiguous block by the following formula.

+ :math:`\text{block's physical address} = \text{block's index} \times 4096 +  \text{base address}`
+ :math:`\text{block's size} = 4096 \times 2^\text{block's exponent}`

Linked-lists for blocks with different size (VF2 Device Tree <memory> node)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
You can set a maximum contiguous block size and create one linked-list for each size.
The linked-list links free blocks of the same size.
The buddy allocator's search starts from the specified block size list.
If the list is empty, it tries to find a larger block in a larger block list

.. _release_redu:

Release redundant memory block
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The above algorithm may allocate one block far larger than the required size.
The allocator should cut off the bottom half of the block and put it back to the buddy system until the size equals the required size.

.. note::
  You should print the log of releasing redundant memory block (via VF2 UART) for the demo

Free and Coalesce Blocks
--------------------------
To allow the buddy system to reconstruct larger contiguous memory blocks on VF2,
when the user frees an allocated block, the buddy allocator should not naively place it back on the free list.
Instead, it must call find_buddy() and merge_iter(), using page frame indices computed from
VF2 Device Tree’s <memory> base addresses.

.. _find_buddy:

Find the buddy
^^^^^^^^^^^^^^

On VF2, compute each block’s page frame index relative to the <memory> region base address.
Then use index XOR exponent to find its buddy’s index. If the buddy lies within the same <memory> region,
merge them into a larger block.

.. _merge_iter:

Merge iteratively
^^^^^^^^^^^^^^^^^
There is still a possible buddy for the merged block.
You should use the same way to find the buddy of the merge block.
When you can't find the buddy of the merged block or the merged block size is maximum-block-size, 
the allocator stops and put the merged block to the linked-list.

.. note::
  You should print the log of merge iteration for the demo.

Basic Exercise 2 - Dynamic Memory Allocator - 30%
=================================================

Your Page Frame Allocator already provides 4 KB-aligned page frames.
The Dynamic Memory Allocator must call the page_allocator API (e.g., p_alloc(1)) to obtain a single page frame
and compute its physical base address. For small allocations (< 4 KB), maintain multiple chunk pools
(e.g., sizes 16, 32, 48, 96 bytes, etc.). Partition each 4 KB page into fixed-size chunks for a given pool.
On each allocation request:
  1. Round up the requested size to the nearest pool size.
  2. If a free chunk exists in the corresponding pool, return it; otherwise, request a new page from the Page Frame Allocator.
  3. Slice the new page frame into chunks and add them to the pool’s free list, then return one chunk.
When freeing a chunk, use its base page frame address to identify which pool it belongs to,
and place it back onto that pool’s free list.

.. admonition:: Todo

    Implement a dynamic memory allocator.
    

##################
Advanced Exercises
##################

.. _startup_alloc:

Advanced Exercise 1 - Efficient Page Allocation on VF2 Device Tree - 10%
=====================================================

Basically, when you dynamically assign or free a page on VF2, your buddy system’s response time should be as quick as possible.
In the basic part, we only care about correctness, but in this section, you must optimize your data structures
so that locating any page frame node takes O(1) time and allocation/free still takes O(log n).

.. admonition:: Todo

   You should allocate and free a page in O(log n), while ensuring any page frame lookup is O(1).

Advanced Exercise 2 - Reserved Memory via VF2 Device Tree - 10%
===========================================

As previously noted in the background, when rpi3 is booted, some physical memory is already in use, not allocable memory blocks must be marked. 
In this task, you should design an API to reserve specific locations.

The following code is a brief example:

.. code:: c

  void memory_reserve(start, end) {
      //…
  }

.. admonition:: Todo

   Design an API `memory_reserve(uint64_t start, uint64_t size)` that marks frames as reserved based on
   VF2 Device Tree <reserved-memory> entries. The Startup Allocator must call this API for each reserved node.


Advanced Exercise 3 - Startup Allocation with VF2 Device Tree - 20%
==============================================
In general purpose operating systems, the amount of physical memory is determined at runtime. Hence, a kernel needs to dynamically allocate its page frame array for its page frame allocator. The page frame allocator then depends on dynamic memory allocation. The dynamic memory allocator depends on the page frame allocator. This introduces the chicken or the egg problem. To break the dilemma, you need a dedicated startup allocator during startup time.

The design of the startup allocator is quite simple. Implement a minimal bump-pointer allocator at startup that does not rely on the page allocator.
This bump allocator must:

  1. Read all physical memory ranges from the VF2 Device Tree’s <memory> node.
  2. Read each <reserved-memory> entry and call memory_reserve(start, size) to mark frames as reserved.
  3. Use the remaining usable memory to allocate space for the Page Frame Array (frame bookkeeping).
  4. After setting up the Page Frame Array, hand over control to the buddy system,
     which will mark the reserved segments as allocated in the frame array.

This bump allocator will only be used during early boot before the buddy system is fully initialized.

.. admonition:: Todo

   Implement the startup allocation.

.. note::
  * Your buddy system must handle VF2’s total physical memory and any holes reported by the Device Tree.
    Read all <memory> regions from the VF2 Device Tree to determine usable segments.
  * All usable memory regions must be used to build the Page Frame Array dynamically via the Startup Allocator.
    Allocate the Page Frame Array out of those ranges, skipping reserved areas.
  * Reserved memory block detection is not part of the Startup Allocator itself.
    Instead, Startup Allocator must call `memory_reserve(start, size)` for each entry in <reserved-memory>.
  * Do not hardcode any physical addresses. All memory ranges (usable or reserved) must be obtained from
    the VF2 Device Tree:

    1. Spin tables or early boot structures, if required by VF2
    2. Kernel image region
    3. Initramfs region
    4. Device Tree blob itself
    5. Any additional platform-specific reserved areas (e.g., CLINT, PLIC)

