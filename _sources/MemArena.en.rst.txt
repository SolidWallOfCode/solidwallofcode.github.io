.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _MemArena:

MemArena
*************

:class:`MemArena` provides a memory arena or pool for allocating memory. Internally the :class:`MemArena` allocates memory in large blocks and allocates memory from these blocks. When :class:`MemArena` is destructed all of the
memory blocks are deallocated. This is useful when the goal is to

*  amortize allocation costs for many small allocations.
*  create better memory locality for containers.
*  deallocate memory in bulk.

:class:`MemArena` has a mechanism for increasing locality by allowing coalescing the arena contents in to a single internal block of memory.

Description
+++++++++++

When a :class:`MemArena` is constructed it can be constructed with no allocated memory or an initial memory block of a
specific size. The :func:`MemArena:reserve` can specific the size of the next internally allocated block if internally
allocation should be delayed until :func:`MemArena::alloc` is called.

To support coalescence and compaction of memory for better locality, the methods :func:`MemArena::freeze` and
:func:`MemArena::thaw` are provided. Calling :func:`MemArena::freeze` locks down all current blocks. After this call any further allocations will be in new internal memory blocks. The first internal block will be large enough to hold all of the currently allocated memory, unless a value is passed to :func:`MemArena::freeze` to set the size explicitly. This enables copying all of the objects in the arena in to a single internal
block. Once this has been done calling :func:`MemArena::thaw` will release the frozen memory blocks.

Other than a freeze / thaw cycle, there is no mechanism to release memory except for the destruction of the :class:`MemArena`. This makes :class:`MemArena` not suitable for a container that allocates variables sized objects (such as something like a :code:`Std::vector`). A container that allocates the same sized objects (such as a :code:`std::map`) can (and should) use a free list.

Reference
+++++++++

.. class:: MemArena

   .. function:: MemArena()

      Construct an empty arena.

   .. function:: MemArena(size_t n)

      Construct an arena with an initial internal block with at least :arg:`n` bytes of free space.

   .. function:: MemSpan alloc(size_t n)

      Allocate memory of size :arg:`n` bytes in the arena.

   .. function:: MemArena& reserve(size_t n)

      Mark the next internal block to have at least :arg:`n` bytes of available space.

   .. function:: MemArena& freeze(size_t n = 0)

      Freeze the existing internal memory blocks and do not allocate from them. If :arg:`n` is zero then for the next allocation request a block sufficiently large to hold all of the contents of the frozen memory will be created. If :arg:`n` > 0 the next internal allocation will have a least :arg:`n` bytes of available space.

   .. function:: MemArena& thaw()

      Free frozen memory.

Internals
+++++++++

Each time a new internal block is needed, it should be larger than the previous block by either the square root of 2 or 2. It might be reasonable to use :code:`IOBuffer` instances for bulk memory, which increase by powers of 2.
