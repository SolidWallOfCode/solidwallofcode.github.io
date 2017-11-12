.. include:: ../common.defs

.. _glossary:

Glossary
********

.. glossary::
   :sorted:

   span
      A contiguous range of bytes on a storage device.

   cache volume
      A logical unit of cache storage.

   stripe
      A subrange of a :term:`span` that is associated with a specific :term:`cache volume`.

   cache object
      A resource identified by a URL. A cache object can have various forms which are called
      :term:`alternates`. These can vary by encoding, format, or other properties. Every alternate
      of an object is considered semantically equivalent.

   alternate
   alternates
      Different variants of a single cache object.

   slice
      A version of an :term:`alternate`. Slices are temporal. Different slices of an alternate are variations of the alternate through
      time. An alternate has more than one slice on when it is active, cache objects in storage have exactly one slice per alternate.

   alternate vector
      A vector of alternate descriptors, instances of :cpp:class:`HTTPCacheAlt`.

   alternate selection
      Selecting a specific alternate for an HTTP object.

   revalidation
   revalidated
      Revalidation is done by sending a conditional request to an origin server to check the status
      of a specific alternate. Once the response is received and process the alternate has been
      revalidated and is presumably fresh.
