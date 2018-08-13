.. include:: common.defs

.. _ats-projects:

Traffic Server Projects
***********************

These are my personal projects for |TS|. These vary from very personal ones to collaborative work. A
large diagram of the projects and their dependencies is `here <_images/ts-projects.png>`__.

==============
Projects
==============

.. toctree::
   :maxdepth: 1

   cache-tool.en
   testing.en
   errata.en
   body-factory.en
   l4r.en
   plugin-coordination.en
   RPC.en

======================
Less Detailed Projects
======================

Most these should be in separate pages but I have not yet had time to properly document them.

Layer 7 Routing
===============

Select upstream target based on the HTTP header information.

.. note::

   Need to consider the ability to bypass upstream selection for non-caching objects (e.g., skip a second caching layer for requests that will always end up at the origin anyway).

Logging tags from plugins
=========================

This has been a desire for a long time but the implementation is a bit tricky with regard to timing,
due to logging being done as the transaction is being terminated and data may have already
disappeared. I have some ideas but those would require some additional infrastructure such as better
arenas.

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

Bijection is a class I wrote for my own product software long ago, but it depends on some
sophisticated Boost library support and so could not be easily ported to |TS|. It was a very useful
class, particularly for configuration work and enumeration support. It is a goal of mine because
building will create a good amount of powerful infrastructure which will be useful in other
situations.

Replace TCL hash maps
+++++++++++++++++++++

There is currently a templated hash map table but it requires externally allocated memory. This has
its benefits in complex situations but is simply annoying for basic hash table use. Adding the
compacting arena to the current :code:`TSHashTable` would yield an easily usable hash table with the
desired memory allocation properties.

Bijection
+++++++++

Given the compacting arena and :code:`TSHashTable` buidling the bijection class is straightforward.
Two hash tables (or a hash table an arra) can easily be constructed on the same elements which is
the essential technique.

Traffic Server Core
===================

Restructure the overridable configs to remove cyclic dependencies. Already discussed with community
and I have a design.

Potentially the "proxy-protocol" extension.

Internal version of the `C++17 filesystem library <http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0218r0.html>`__.

Fix :code:`TSNetConnect` to have options instead of a multiplying set of API calls.

Make remap rules be pure first match (long term).

The crypto hash support needs to be cleaned up.

Add log tags to dump transaction headers in full. This would be useful for the operations teams. The
idea is a custom log is setup that logs the full headers but is filtered by error return codes. The
result is a log of full headers only for failed transactions. We had originally looked at doing this
with a plugin but Dan Xu discovered that it would be easier to add these tags and use the existing
logging mechanisms.

Use :code:`std::chrono`.

Prevent restart on cache failure (bad disks).

Transaction arenas for plugin use.

Pending event counting at base event loop / continuation.

Look at allocating SDK Handles from the transaction arena instead of a global allocator, to avoid
cleanup issues.

TLS Extensions
++++++++++++++

It would be interesting in terms of L4 routing to be enable TLS clients to send a |TS| specialized
TLS extension to provide addition L4 routing information or other control data. This needs to be
design carefully to avoid security issues but I think it could be very powerful.

Cache API Toolkit
=================

This is a restructuring of how the cache to enable fine grained control by plugins. Put in a
reference to the summit presentation on this.

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

.. Files referenced but not to be in a TOC.

.. toctree::
   :hidden:

   tsconfig-lua.en.rst
