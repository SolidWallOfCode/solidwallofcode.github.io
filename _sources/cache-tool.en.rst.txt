.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _cache tool:

.. toctree::
   :hidden:

   command-registration.en

Cache Tool
**************

CacheTool is a command line tool for interacting with the |TS| cache. CacheTool started as a side
project to not only explore the |TS| cache but as a test bed for various other technologies of
interest to me, some of which are not really cache related. The goal is for the eventual release
of these other technologies to open source as they mature. My view was the ancillary software would
be of better quality if validated in actual use first.

*  `TS.Scalar <https://docs.trafficserver.apache.org/en/latest/developer-guide/internal-libraries/scalar.en.html>`_: a template for creating quantized integral types. This has been contributed to open source.
*  `TextView <https://docs.trafficserver.apache.org/en/latest/developer-guide/internal-libraries/memview.en.html>`_: read only view of string data. This has been contributed to open source.
*  Better :ref:`command line parsing <command registration>` for action oriented tools like CacheTool and :program:`traffic_ctl`.
*  File system path and file handling class.
*  New data structures for the cache, as part of a larger scale redesign.
*  Improved stripe allocation algorithms.

In addition to being a test bed, Cache Tool was intended to do useful work. An early feature, stripe
re-allocation, was built to get around a bug in |TS| where a disk that was replaced would not
actually be used by the cache. This was originally a request from the Oath operations team but it
was sufficiently useful to another member of the community that I was persuaded to make CacheTool
public before it was even alpha quality.

Some of the intended features of CacheTool are

*  Fix stripe allocation for unused disks.
*  List stripe data for a cache.
*  Clear / reset cache disks independently.
*  Provide cache directory analysis, as is done by passing :code:`--command check` to :program:`traffic_server` now, but with sufficiently low resource requirements it can be run on live system.
*  Cache directory diagnostics.
*  Cache scanning

   *  Scan to purge.
   *  Statistical analysis.

*  Cache repartitioning.

   *  Store additional information about cache configuration.

      *  More efficient and reliable examination.
      *  Required to repartition, if values are always computed there is a chicken and egg problem.

   *  Update stripes to shift storage between directory and content (change effect average object size).
   *  Maintain backwards compatibility and some forward compatibility. In particular it must be
      possible to run 8.0 on a box and revert to 7.X without clearing the cache.

Usage
=====

CacheTool has an action oriented interface with commands and subcommands (and possibly
subsubcommands, etc.), modeled on :program:`traffic_ctl` but with a much better internal implementation.

``list``
   List information about the cache.

   ``stripes``
      List per stripe information.

``alloc``
   Allocate cache space to stripes.

   ``free``
      Allocate all free space to volumes.

   ``sim``
      Simulate a full allocation of space to volumes.

``dir``
   Stripe directory operations.

   ``freelist``
      Scan and fix segment free lists in the stripe.

   ``stats``
      Print analytical data about the stripe directory.

``span``

   ``clear``
      Clear a span. All space in the stripe is consolidated in a single free block.
      This command takes a list of span identifiers, each of which is cleared.

      ``all``
         Clear all spans.

Implementation
==============

This section has internal implementation notes.

Directory Free Lists
--------------------

Each segment in a directory has a free list. It is possible for this to become corrupted and have a
loop in the list. There is some logic which can be conditionally compiled in the |TS| core that will
place a maximum length on the free list. Upon detection an error is generated.

The CacheTool will detect free list loops more directly by using a :code:`std::bitset` to mark
directory entries that have been encountered while walking the free list. If a duplicate is detected
(indicating a loop) then rather than attempt any sort of complex repair, the CacheTool will clear
and then reconstruct the free list in a way similar to how the free list was constructed during
stripe initialization. This is primarily constructing the list so that directories at higher layers
in the buckets are earlier in the free list.

Rate Limiting
-------------

While in general the CacheTool will be run while |TS| is offline it will frequently be desirable to
be able to run CacheTool while |TS| is also running. To make this possible CacheTool will need to be
able to rate limit its disk reading. This will be specified by placing a limit on the number of
bytes per second that will be read. It is not expected this limiting will be exact but it should be
reasonably close. An possible implementation is to start with the per second limit as a read limit
and then after the read completes, subtract the size of the read and add the limit value for each
second that has passed. It may also be good to limit the addition to a maximum number of seconds to
further restrict I/O when the disks are under excessive load.

Future Work
===========

This section contains some of the larger work planned for the CacheTool.

Content Scanning
----------------

A key feature is the ability to scan content stored in the cache. This is extremely expensive in
terms of disk I/O. This is of less concern if the system is offline, in which case CacheTool can
consume all availale disk I/O. Even in that case, however, the amount of data to be read can still
large enough to make the overall runtime too large if the data is read in a simplistic fashion.

The primary purpose of cache scanning is to extract the request and response headers. All of this
data is contained in the first fragment of the cached object. This is noted in the directory entry
which means disk reads can be restricted to only these fragments rather than a full scan of the
content area.
