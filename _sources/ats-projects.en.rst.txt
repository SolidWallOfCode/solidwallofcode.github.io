.. include:: common.defs

.. _ats-projects:

Traffic Server Projects
***********************

Diagram.

.. graphviz:: dot/ats-projects.dot

.. toctree::
   :maxdepth: 1

   buffer-writer.en
   tsconfig-lua.en
   cache-tool.en
   testing.en
   errata.en

Layer 7 Routing
===============

Select upstream target based on the HTTP header information.

.. note::

   Need to consider the ability to bypass upstream selection for non-caching objects (e.g., skip a second caching layer for requests that will always end up at the origin anyway).

Forwarded Header
================

A basic implementation was done for our fork of ATS 5.3.x. This suffices for internal use for now, but for open source it will have to be done better. I did some design based on Walt's work and then Walt, Leif, and I had a discussion on future directions for this work. That breaks down in to several steps.

*  Fixed buffer printer class, which takes a buffer and then supports type safe printing on it. See :ref:`buffer writer`.
*  A subclass of the fixed buffer printer which takes a template argument to allocate local (stack) storage for the buffer.
*  Update the current Forwarded header logic to use these classes.
*  More flexible configuration.

Body Factory Cleanup
====================

The core logic used to generate proxy replies is convoluted and fragile. It has needed to be cleaned up for a long time both to make it more robust and maintainable but also to add features that are currently hard to do. There are also arbitrar internal limits which are entirely artifacts of the implementation.

The basic plan is to extend the fixed buffer printer classes to have one that takes an :code:`MIOBuffer` as its output. This would make it basically equivalent to C++ I/O stream operators but using |TS| core mechanisms. Then a new "Format" class would be done which processes the current log style format strings in a generic and modular way. The body factory would be updated to use these new classes.

Logging tags from plugins
=========================

This has been a desire for a long time but the implementation is a bit tricky with regard to timing, due to logging being done as the transaction is being terminated and data may have already disappeared. I have some ideas but those would require some additional infrastructure such as better arenas.

C++ ABI to Core
===============

There is currently a C++ API for plugins but this is really a wrapper on the C API. It would be good
to be able to have actual C++ APIs in to the core. There are a number of core features that should
be made available to plugins, such as the network address handling code and the string view classes.
The primary problem is the fragility of the C++ ABI, some mechanism would be needed to deal with
this.

Plugin C++ API
==============

The plugin C++ API needs a lot of polishing and updating.

Bijection
=========

Bijection is a class I wrote for my own product software long ago, but it depends on some sophisticated Boost library support and so could not be easily ported to |TS|. It was a very useful class, particularly for configuration work and enumeration support. It is a goal of mine because building will create a good amount of powerful infrastructure which will be useful in other situations.

Compacting Arena
++++++++++++++++

Memory arena support should be formalized in to a support class. Having a better string arena for
transactions would be a clear improvement in memory handling. The compacting arena is one that does
standard memory arena support but also enables compacting / coalescing its in use memory in to a
single block. This is very useful for data constructed at process start time and then not updated.

Replace TCL hash maps
+++++++++++++++++++++

There is currently a templated hash map table but it requires externally allocated memory. This has its benefits in complex situations but is simply annoying for basic hash table use. Adding the compacting arena to the current :code:`TSHashTable` would yield an easily usable hash table with the desired memory allocation properties.

Bijection
+++++++++

Given the compacting arena and :code:`TSHashTable` buidling the bijection class is straightforward. Two hash tables (or a hash table an arra) can easily be constructed on the same elements which is the essential technique.

RPC Refactoring
===============

The current RPC mechanism used to communicate between command line tools and :code:`traffic_manager` is rather a mess and implemented in a assymmetric way. It should be refactored in two steps.

*  Move the RPC logic to separate library.
*  Make the RPC symmetric / bidirectional.

As a side project, there is currently a roughly 5 second delay in communications for no apparent reason. As best as I can tell it is a deliberate pause to avoid :code:`epoll`. This should be fixed.

Traffic Server Core
===================

Restructure the overridable configs to remove cyclic dependencies. Already discussed with community and I have a design.

Event loop changes - put in our 5.3.x fork but never contributed to open source. This will be necessary relatively soon in the internal 7.1.x codebase. Unfortunately this requires the TS-4265 thread initialization changes first.

Potentially the "proxy-protocol" extension.

Internal version of C++17 :code:`filesystem` library.

Internal version of C++17 :code:`string_view` library and |TS| specialized version :code:`StringView`.

Fix :code:`TSNetConnect` to have options instead of a multiplying set of API calls.

Make remap rules be pure first match (long term).

The crypto hash support needs to be cleaned up.

Look at using `TBB <https://www.threadingbuildingblocks.org>`_ for thread safe containers.

Add log tags to dump transaction headers in full. This would be useful for the operations teams. The
idea is a custom log is setup that logs the full headers but is filtered by error return codes. The
result is a log of full headers only for failed transactions. We had originally looked at doing this
with a plugin but Dan Xu discovered that it would be easier to add these tags and use the existing
logging mechanisms.

TLS Session re-use updates. Currently TLS session re-use is done by having the plugin directly
control the openSSL callbacks. This leads to conflicts and also requires some custom core changes
used only internally to update some core state. It would be much better to have the |TS| core
control the openSSL callbacks and farm out the work to plugins as needed. This would not only reduce
plugin conflict and remove the need for custom core changes but also would let our plugin take
advantage of the openSSL session table while still sharing them across a pod. Additionally we should
look at removing the use of MDBM both for reliability and the potential to contribute the TLS
session resuse plugin to open source.

Use :code:`std::chrono`.

Prevent restart on cache failure (bad disks).

Transaction arenas for plugin use.

Pending event counting at base event loop / continuation.

Dynamic Cert Loader
===================

An open source plugin that does dynamic certificate loading / management. This is to avoid a huge delay at process start and instead load certificates as needed. We are likely to be using a lot more distinct certificates in the future and this is also a feature that will be important in comparing to VDMS. There are two phases for this

* Load certificates on demand, and release unused certificates.
* Dynamically generate impersonation certificates.

.. note::
   A dynamic certificate loader could also make rotating certificates easier. Although |TS| can be provided with updated certificates will running this will reload all certificates which is even worse than the start up problem. In addition a certificate loader could respond to plugin messages or check files times to reload only changed certificates, or permite revocation without a reload.

Plugin Coordination
===================

There is a consistent need for coordination among plugins. At one point (Feb 2016) I had a working prototype of plugin priorities based on Y! 5.3.x which assigned priorities to plugin callbacks along with an API to manipulate not just the current plugins priorities but that of other plugins. That would be very useful although priorities of themselves are only one approach to this problem. Based on that work I have a technology ladder to climb to provide this capability.

Modular hook dispatch.
   Create a class to handle the hook dispatching to provide a level of consistency needed later.
Continuation tracking
  Associate every continuation with a "root" plugin. This would be very valuable in debugging plugin problems and would result in significantly less developer time spent with faster results for customer support.
TS API to probe hooks for callbacks.
   Allow plugins to check the callbacks on a hook.
TS API to manipulate callbacks on hooks
   Allow plugins to alter the callbacks on a hook, either in terms of priorities, dependency, or direct access by plugin.

HTTP/2 Outbound
===============

|TS| should be able to use HTTP/2 outbound. This will require some restructuring of the internal classes used to model outbound connections (similar to the restructuring needed for inbound HTTP/2 as expressed in TS-3612).

jemalloc and memory allocators
==============================

We need to proceed with work on testing :code:`jemalloc` and its interaction with |TS| memory allocation.

TLS
===

openSSL 1.1
   |TS| needs to be compatible with the openSSL 1.1 library. This is mostly done, it should be primarily verification at this point.

TLS/1.3
   This is a soon to be standard. We should start planning for it.

Cache API Toolkit
=================

This is a restructuring of how the cache to enable fine grained control by plugins. Put in a reference to the summit presentation on this.

Live Restart
============

A long running request is to be able to do a live restart of |TS|. The mechanism would be

*  Start new |TS| process.
*  New process starts accepting connections.
*  Old process stops accepting connections.
*  Old process shuts down.

   *  When all inbound connections have terminated.
   *  When there are less than a specified number of inbound connections.
   *  After a specific amount of time.
   *  When explicitly requested by the administrator.

The main difficulty for this is handling the cache. To some extent the cache would need to be
multi-process. To make this more feasible the access would be single writer and the control of
writing would pass from the old process to the new process. This may mean terminating cache writes
in the old process.
