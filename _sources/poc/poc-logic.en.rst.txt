.. include:: ../common.defs
.. default-domain:: cpp

Cache Logic / Operations
************************

Cache Startup
=============

.. uml::

   @startuml

   hide empty members
   hide empty methods
   skinparam defaultTextAlignment left

   class CacheProcessor
   class DiskInit
   class CacheDisk
   class VolInit
   class Vol

   CacheProcessor --> DiskInit : //create//
   DiskInit --> CacheDisk : [1] open()
   note on link : Start I/O for Span Header
   CacheDisk --> CacheDisk : [2] openStart()
   note on link : Validate Span Header
   CacheDisk -d-> CacheDisk : [3] openDone()
   CacheDisk --> CacheProcessor : [4] diskInitialized()

   CacheProcessor --> Cache : [5] open()
   Cache --> VolInit : [6] mainEvent()
   VolInit --> Vol : [7] init()

   @enduml

State machine.

.. graphviz:: cache_sm.dot

Lock Waiting
============

To avoid the current issues with excessive event flows when many :class:`CacheVC` instances are
waiting for locks along with many failed lock attempts, the :class:`Vol` and :class:`OpenDirEntry`
maintain lock waits lists. These are organized differently due to the different natures and access
patterns for the classes. In both cases the wait lists are lists of |TS| events that refer to
:class:`CacheVC` instances. This indirection is critical in order to avoid problems if a
:class:`CacheVC` is destroyed while waiting for a lock. It can then cancel the event without leaving
dangerous dangling pointers. The lock owner can then discard canceled events without any further
dereferencing. This is the same logic used by the core event loop.

:class:`Vol` uses thread local storage to store wait lists. For each thread there is a vector of
event lists. Each :class:`Vol` has a numeric identifier which also serves as its index in the
vector. This is possible because the set of :class:`Vol` instances is determined by the cache
configuration and therefore fixed at process start time. Content for :class:`Vol` locks is, based on
experimental measurements, the lock with the highest contention and therefore is worth special
casing for performance. The locks are spread among threads for a few reasons.

Load Distribution
   The number of waiting :class:`CacheVC` instances can be large and having them all dispatch in one pass could easily introduce excessive latency in to whichever event loop gets unlucky. Conversely breaking up thundering herds somewhat from the cache point of view is also beneficial.

Thread Consistency
   Waiting :class:`CacheVC` instances can be dispatched on their preferred thread. This is more important for :class:`Vol` interaction as those are much close to the HTTP state machine which prefers to always run on a single thread.

:class:`OpenDirEntry` uses a single global atomic list of events per instance. It is not feasible to
use thread local storage because instances of :class:`OpenDirEntry` are created and destroyed
frequently and the corresponding number of instances fluctuates over a large range. The load
distribution issue is of lesser importance because of the (usually) much larger number of instances
which spreads the load when the instances become available. It is unfortunate this causes
:class:`CacheVC` instances to dispatch on other threads but the :class:`OpenDirEntry` interactions
tend to be less closely coupled to HTTP state machine instances.

Write Operations
================
