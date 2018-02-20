.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _MemArena:

MemArena
*************

:class:`MemArena` provides a memory arena or pool for allocating memory. The intended use is for allocating many small chunks of memory - few, large allocations are best handled independently. The purpose is to amortize the cost of allocation of each chunk across larger allocations in a heap style. In addition the allocated memory is presumed to have similar lifetimes so that all of the memory in the arena can be de-allocatred en masse. This is a memory allocation style used by many cotainers - elements are small and allocated frequently, but all elements are discarded when the container itself is destroyed.

Description
+++++++++++

:class:`MemArena` manages an internal list of memory blocks, out of which it provides allocated
blocks of memory. When an instance is destructed all the internal blocks are also freed. The
expected use of this class is as an embedded memory manager for a container class.

To support coalescence and compaction of memory, the methods :func:`MemArena::freeze` and
:func:`MemArena::thaw` are provided. These create in effect generations of memory allocation.
Calling :func:`MemArena::freeze` marks a generation. After this call any further allocations will
be in new internal memory blocks. The corresponding call to :func:`MemArena::thaw` cause older
generations of internal memory to be freed. The general logic for the container would be to freeze,
re-allocate and copy the container elements, then thaw. This would result in compacted memory
allocation in a single internal block. The uses cases would be either a process static data
structure after initialization (coalescing for locality performence) or a container that naturally
re-allocates (such as a hash table during a bucket expansion). A container could also provide its
own API for its clients to cause a coalesence.

Other than freeze / thaw, this class does not offer any mechanism to release memory beyond its destruction. This is not an issue for either process globals or transient arenas.

Reference
+++++++++

.. class:: MemArena

   .. function:: MemArena()

      Construct an empty arena.

   .. function:: MemView alloc(size_t n)

      Allocate an :arg:`n` byte chunk of memory in the arena.

   .. function:: MemArena& freeze(size_t n = 0)

      Block all further allocation from any existing internal blocks. If :arg:`n` is zero then on the next allocation request a block sufficiently large to hold all current allocations is created, otherwise the next internal block will be large enough to hold :arg:`n` bytes.

   .. function:: MemArena& thaw()

      Unallocate all internal blocks that were allocated before the last called to :func:`MemArena::freeze`.

Internals
+++++++++

Each time a new internal block is needed, it should be larger than the previous block by either the square root of 2 or 2. It might be reasonable to use :code:`IOBuffer` instances for bulk memory, which increase by powers of 2.
