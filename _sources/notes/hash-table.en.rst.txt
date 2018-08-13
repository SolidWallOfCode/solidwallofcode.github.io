.. include:: ../common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _hash-container-notes:

********************
Hash Container Notes
********************

Currently there is a plethora of hash table in the |TS| core. One the initiatives from the Cork was
to attempt to reduce this number. Before doing that successfully it is necessary to understand the
uses cases inside |TS|.

=========
Use Cases
=========

Simple Hash Table
   Use of a hash table for rapid lookup of stored items.

Concurrent Hash Table
   The simple hash table case with the additional requirement concurrency. In general this means at
   most one writer and an arbitrary number of reads without explicit locks. Multiple writers are
   handled by an explicit lock which is required only for table updates, not for searching or
   reading.

Multi-index hash tables
   In a few cases the items stored in the table need to be accessed via multiple distinct keys. The
   archetypical example is the upstream session table which needs to be keyed by FQDN and IP
   address.

External memory management
   Because of the heavy use of proxy and class allocators, it is sometimes the case that memory and
   life time management of objects is independent of the hash container. Again, the upstream session
   table is archetypical in that sessions are put in and removed from the hash container, the
   container should not create or delete these objects. While it would be possible to use a
   container of smart pointers, this means allocating and deallocating nodes in the container for
   inserts and deletes, in contrast to an intrusive container which does not require such memory management.q

A side property which is not required but is very conventient is the ability to use a member of the
object as a key. While the key can be split off, this tends to tie the object to the container.
Alternatively the key can be duplicated but if key requires local storage this requires another
memory allocation for each object created.

=======
Designs
=======

There are two basic approaches to supporting hash containers in |TS|.

Standard Template Library
=========================

The Standard Template Library provides the containers :code:`std::unordered_set` and
:code:`std::unordered_map`. These suffice for basic hash container use. The main concerns with the
STL containers are

#. Object lifetimes and memory management are tied to the container. This does not interact well
   with proxy allocators for reasons noted above.

#. Relatedly, memory management is "invisible" and therfore tempting to use in places where
   allocations should be avoided. It is easy to use STL containers in ways that have very bad memory
   use characteristics.

The primary concern here, as I understand it, is that use of STL containers for specific purposes
will encourage the same use in general, leading to significant performance degradation. This is not
an idle concern - I am currently involved inside Oath with another project, a plugin for |TS|, which
manages to consume as much CPU as the rest of |TS| put together due to uninformed choices in using
STL containers.

As noted previously, the memory management issues could be worked around using smart pointers.
However this would create a significant amount of additional memory management activity as the nodes
in the container will need to be allocated and released with every insert and removal.

Based on research in to allocators for STL containers, it does look like this is possible. A major
change in this area for C++11 is allocators can now be stateful. This is critical to prevent memory
migration between two containers, e.g. if memory is allocated from an allocator for container A, it
is not used for container B. However, currently this requires an update to :code:`MemArena` which is
blocked.

Intrusive Containers
====================

|TS| has to a large extent rolled its own containers, primarily intrusive ones where the objects
directly support the containers. This is likely due to the style of memory management where objects
are allocatedand released independently of containers. This was originally a set of what became
legacy containers in the header file "lib/ts/Map.h". These lacked not only documentation but any
compliance with STL standards, such as iterators, and handled only pointers. The containers also
depended on the :code:`Vec` class which is being phased out. For this reason, and to support
multi-indexing, the class :code:`TSHashTable` was created. :code:`TSHahhTable` had the advantages of

*  Contain an arbitrary type :code:`T` as long as it had intrusive list support.

*  Control of the key.

*  Control of bucket expansion policy.

*  Iteration.

These features make the container much easier for a standard C++ programmer to use as it enables
following standard STL use patterns.

The ability to select the key is valuable for performance as well because :code:`IntrusiveHashMap`
takes care to never default construct a key, but only initialize them. This means the type used by
the container may be a reference or pointer to the actual key. For example, if the key is a
:code:`std::string` then the key used for the container can be :code:`const std::string &` or even a
:code:`std::string_view`, with the result that searching the container does not require
instantiating a :code:`std::string` and the attendant memory management. There have been attempts to
work around this by using STL containers and making the key a reference or even a
:code:`std::string_view` in to the object but these have failed with obscure memory corruption
issues. At present this is not a viable option.

Selecting the key also enabled the multi-index capability in that each index can select a different key.

With the switch to C++11 and then C++17, it seemed reasonable to update this class to take advantage
of the new language features and become more STL compliant. This resulted in
:code:`IntrusiveHashMap` which has the advantages

*  Use of :code:`IntrusiveDList`, a pure C++ class, instead of the C macro based linked lists used previously.

*  STL compliant support for ranges.

*  Full iterator support, replacing the psuedo-iterator :code:`TSHashTable::Location`.

*  Simplified specification, reducing the number of elements required in the descriptor type.

:code:`IntrusiveHashMap` has been through several iterations and example uses have be provided in
pull requests. The intention is for :code:`IntrusiveHashMap` to completely replace
:code:`TSHashTable` and uses of the other hash maps in "lib/ts/Map.h". If the necessary support in
:code:`MemArena` (currently blocked) can be made available then :code:`IntrusiveHashMap` could be
used to replace the TCL hash tables.

:code:`IntrusiveHashMap` works well with the standard allocation style used in |TS|. This will make
it relatively easy to replace current uses of internal legacy hash containers with

:code:`IntrusiveHashMap` satisfies the specialized requirements for multi-indexed containers. Given
that, in my view, we will want this class for this reason, it seems a low marginal cost effort to
use the class in the other situations mentioned above. Not doing so will not remove the need for
this class.

===========
Concurrency
===========

Concurrency is hard to get right and fast. Currently there are custom concurrent hash containers in
|TS|. These use the technique of splitting locks per bucket, or using a pool locks [#]_. The
alternative recommended at Cork is to use a well seasoned third party concurrent hash container. The
three options are

#. `Thread Building Blocks <https://www.threadingbuildingblocks.org/>`__

#. `Concurrent Data Structures <http://libcds.sourceforge.net/>`__

#. `Concurrency Kit <http://concurrencykit.org/>`__

Each of these has issues from the |TS| point of view but, in my opinion, any would be a improvement
over the current state of concurrent containers.

Thread Building Blocks
   This is sponsored by Intel, but it is a heavy weight solution designed for a different domain
   (compute intensive data processing) than that of |TS| (event driven I/O processing). Use of TBB
   depends on being able to use just the concurrent containes and not most of the library. TBB is
   also under development and would require periodic updates.

Concurrent Data Structures
   This is a nice set of concurrent containers although the documentation is a bit thin. The main
   problem here, in my view, is its dependency on thread support. It is unclear to me why this is
   necessary, but the main thread and each thread that uses the library must explicitly "attach" to
   the library in the thread. The development status is unknown.

Concurrency Kit
   This is a very mature, well tested and fast set of concurrency containers. The problem here is it
   is written in C and not amenable to easy conversion to C++11. In fact, the C code is such that it
   will not even compile as C++ without extension re-skinning. It has been discussed to start a
   "CK++" project to do this, which was endorsed by the Concurrency Kit author. The base C version
   is a completed project, there is no active development.

While the various merits and costs can be debated, the reality is likely we will adopt which ever of
these is first brought in to compatibility with |TS|. My personal preference is for CK++, the
reskinning of Concurrency Kit.

.. rubric:: Footnotes

.. [#]

   In some sense these aren't different - if the mutex in the pool is selected by key hashing it is
   effectively the same as a mutex per bucket.
