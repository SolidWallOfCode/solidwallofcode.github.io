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
interest to me, some of which are not really cache related. The goal was for the eventual release
these other technologies to open source as they matured. My view was the ancillary software
would be of better quality if validated in actual use first.

*  `TS.Scalar <https://docs.trafficserver.apache.org/en/latest/developer-guide/internal-libraries/scalar.en.html>`_: a template for creating quantized integral types. This has been contributed to open source.
*  `StringView <https://docs.trafficserver.apache.org/en/latest/developer-guide/internal-libraries/memview.en.html>`_: read only view of string data. This has been contributed to open source.
*  Better :ref:`command line parsing <command registration>` for action oriented tools like CacheTool and :program:`traffic_ctl`.
*  File system path and file handling class.
*  New data structures for the cache, as part of a larger scale redesign.
*  Improved stripe allocation algorithms.

In addition to being a test bed, Cache Tool was intended to do useful work. An early  feature, stripe
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
subsubcommands, etc.), modeled on :program:`traffic_ctl`.

``list``
   List information about the cache.

   ``stripes``
      List per stripe information.

``alloc``
   Allocate cache space to stripes.
