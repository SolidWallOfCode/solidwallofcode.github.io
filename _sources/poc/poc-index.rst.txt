.. include:: ../common.defs

Partial Object Caching
****************************

Partial object caching is the ability to cache and serve range requests. Doing this requires a major restructuring
of the cache logic. This means this is not a feature that can be turned on or off, even at build time.

Once these changes are made some other significant features become easier to include in
the overall work. These features include

*  Stale while revalidate support in the core.
*  Collapsed forwarding.

.. toctree::
   :maxdepth: 2

   poc-logic.en
   poc-data-structures.en
   cache-types.en
   cache-glossary.en

Archives
========

These are historical documents relating to partial object caching. These are documents written in
the past and not yet updated. There are here primarily for reference and in the hope that I will be
able to update them in the not distant future.

.. toctree::
   :maxdepth: 2

   archive-partial-object-caching.en
   archive-partial-object-caching-prototype.en
