Partial Caching Initial Design
==============================

.. note:: This is obsolete and is kept for historical reasons. It was written before the actual implementation.

Partial object caching is really the caching of range requests for the same object as a single object. It is now common for large objects that should be cached to never be retrieved in full but only via range requests. Currently ATS will never cache any part of such an object.

This design should enable caching range requests as part of a single object with only minimal changes to the cache internals.

Basics
------

We want this design to have a few essential properties.

* Data requests to the origin server must not be much larger than the original client request.
* Range requests for an object may occur in any order with any overlap.
* Serve ranges requests from cache if the all of the requested range is already in the cache.
* Server content from cache for a range request if possible.
* Minimal additional overhead of data and computation.

Normally a large object is stored as an ordered list of fragments each of roughly the same size. For a partial object we fix this size for a particular object. Different objects may have different fragment sizes but we require all fragments of specific partial object to be identical in size, except possibly for the last fragment. This is normally the case for all multiple fragment objects although it is not an explicit requirement. The only change here is that we will need to store the fixed fragment size in the metadata of the first fragment per alternate for a partial object.

Given that fixed fragment size we then (conceptually) divide the object in to ranges of that size and call those "slices". We can then store each slice in a single ATS fragment. We say a slice is *touched* by a range request if any byte of that request is in the slice. Because we fix the slice size we can skip untouched slices without losing the ability to insert fragments for those slices later.

.. note::
    Because fragments are computationally chained it is not possible to insert a fragment. The ordering of fragment cache ids is fixed by the cache id of the earliest fragment.

Range requests are processed by adjusting the size seen by the origin server to always be an integral multiple of the slice size with a offset that is also an integral multiple of the slice size. This can mean expanding or shrinking the request dependent on how it intersects already cached data. The altered size is chosen to be the minimum size that covers all of the untouched slicess while satisfying the size and offset requirements. If this yields a size that covers the entire object then the object will not be partially cached but will be retrieved in full.

The client always recieves the originally requested range with the appropriate response header.

With this change we then use the fragment offset table to store the fragment presence. Because we have adjusted the request to the origin server a fragment will always be either all present or not present at all in the cache. Because the fragment size is fixed, we can compute the fragment offsets without having to store them, therefore we can use them purely as a marker while still allowing normal range acceleration operation if the requested range is covered by cached fragments.

Another minor change is that we permit an empty earliest fragment if the first range request does not touch that fragment. This is necessary because the cache logic assumes the earliest fragment is also closes to the stripe write cursor and will therefore be overwritten before any other fragment. This allows validity checking before any cache data is served. We must preserve that and so will write an empty earliest fragment even if it is not retrieved from the origin server. The current logic always reads the earliest fragment (if distinct from the first fragment) so doing this keeps the same logic for checking a partial object. We will need to make a note in the first fragment metadata about this so that the fragment table offsets can be done correctly (in effect this will shift the indices by one). The earliest fragment is a partial fragment (not the slice size) *if and only if* it is also the last fragment. That is, the entire object fits within that fragment. Otherwise the earliest fragment is either of length zero or the slice size.

As range requests are processed for a partial object additional fragments are written and the first fragment is updated. This keeps all of the content fragments bookmarked between the earliest fragment and the first fragment as for full objects. It also allows the fragment offset table to be extended as needed if fragments beyond the current end are retrieved.

Transforms
----------

Transforms present some difficulties for this design. Changes in size can be accommodated to a moderate extent - fragments can be written out as smaller or larger than the fixed fragment size because the fixed size is based on the original content as seen by the origin server. There is a maximum fragment size and transformed data cannot be written to multiple fragments which creates an upper bound on expansion. This is only a problem for storing post-transform content.

The major problem is the nature of the range request which may provide data out of context for any transformation. This problem is not specific to partial object caching and so may be amenable to whatever procedures are used currently to deal with this. A related problem is clients that are tuned to avoid this problem by fetching on boundaries relevant to that client. Because request ranges are modified before being presented to the origin server a corresponding transform plugin may not be able to correctly process the incoming data.

I think this is not a problem if the content is stored pre-transform. In that case we can always feed the transform the same range as the original request. Storing post-transform content remains challenging.

Implementation
--------------

One new class that will be needed is a bidirectional ``CacheVC``. The current class executes either as a source (cache read) or a sink (cache write). For partial objects it need to function as both a producer and a consumer. It will need to splice data from the origin server with content from the cache in the cases where the client request covers both touched and untouched fragments. This requires it to act as a pass through between the origin server network VC and the client network VC. It will also have to handle cache reads and writes, to pull existing cache content for the client and store incoming data from the origin server. My current intention is to read cache content only for the first and last segments of the client request and use origin server data for the middle. If the client request is entirely in the server response then no cache data will be used.

Notes
-----

.. note::
    An open question is the handling of 304 responses and object refresh in general. If the client makes a range request that touches previously untouched fragments we can't attach an ``If-Modified-Since`` field to it. For a rane request that can be satisfied from data in the cache, I don't know if we can attach an ``If-Modified-Since``. If that is possible then I presume we would treat the entire object uniformly, that is all bytes are valid or stale, not just the fragments touched by the client request.

    A related issue is if we get different expiration data for different requests to the origin server. Should the object be updated with the new expiration data or keep the original?

.. note::
    A serious issue with caching range requests is tracking the valid bytes. It is to avoid this problem that we adjust the size of the range request to the origin server. By limiting it to being more than than 2 fragment sizes larger than the original request we avoid excessive network traffic beyond the client request while guaranteeing we need only track the fragments, as is already done for full objects.

.. note::
    Although we must rewrite the first fragment on every request that adds fragments to the partial object we still need to be a bit careful about the full size so that we don't write a partial fragment in the false belief that it is the last fragment.

.. note::
    It is easiest to  use the target fragment size for the fragment size of a partial object but this is in no way required. Currently I don't think it is a useful feature to be able to tune this independently but it would require very little extra work to implement. The main concern would be the additional complexity of configuration as seen by the user, although we could default it to zero which would mean "use the target fragment size". Then it would need to be configured only by those who had an interest in doing so.
