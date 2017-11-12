.. Licensed to the Apache Software Foundation (ASF) under one
   or more contributor license agreements.  See the NOTICE file
   distributed with this work for additional information
   regarding copyright ownership.  The ASF licenses this file
   to you under the Apache License, Version 2.0 (the
   "License"); you may not use this file except in compliance
   with the License.  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing,
   software distributed under the License is distributed on an
   "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
   KIND, either express or implied.  See the License for the
   specific language governing permissions and limitations
   under the License.

.. include:: ../common.defs

.. _partial_object_caching_architecture:

Partal Object Caching
*********************

Introduction
~~~~~~~~~~~~

Partial object caching is the capability to store partial content in the |TS|
cache. This requires large architectural changes as the original design had many
implicit dependencies on all content being present if an object was cached.

Architectural changes
=====================

The primary change is that all content, requests, and responses are treated as
ranged requests internally with a full request being just an optimized special
case where the range is the entire object.

Decoupling reading and writing
------------------------------

Partial content causes a decoupling between the serving of cache
content and the writing of cache content. Given a partial cache object and a
range request from a user agent, there may be little relationship between what
is requested from the origin and what is served to the user agent. In particular
served content can be an arbitrary mix of previously cached content and fresh
content, dependent on the specific history of requests for the object. This is a
major change in that previously there was either a reader VC that served content
from the cache, or a writer VC that wrote to the cache in parallel to serving it
to a user agent.

Object Contention
~~~~~~~~~~~~~~~~~

When an object is found in cache or a new cache object is created an in memory record of that object is allocated, an instance of :cpp:class:`OpenDirEntry`. This tracks the state of the object. In the object, and therefore in the ODE (:cpp:class:`OpenDirEntry`) there is an :term:`alternate vector` which is a set of :term:`alternates` for the object. Initially this is an empty vector (for newly created objects) but can contain multiple entries for the object. The alternate vector gets updated (update of a specific alt or the addition of a new alt) from the response headers of a request. When such a request is made to the origin, this state is referred to as *pending alternate vector update*. This state lasts until either the response headers arrive (and the alternate vector is updated) or the origin connection fails. By design, only one alternate vector update can be pending. While one is pending no other request can be proxied to the origin for that object. This was the subject of much thought but currently it seems the best approach, primarily because doing otherwise creates complexity without apparent benefit. It is important to note this does not mean only one request to an origin for an object can be outstanding - once the response headers are received a new request can be made. Additional request should not be any faster than waiting for a request that is already on the wire except for connection issues with the origin server.

If, after a cache read hit, there is a pending alternate vector update, the transaction will wait for a configurable number of milliseconds for the update to complete. If this times out the transaction must do one of the following -
   *  Go direct to origin without caching.
   *  Fail the transaction with an error.
   *  Serve stale content (if available).\\aspx

Note that serving stale content is not always an option as it may not exist (e.g., the transaction is waiting on a request for a new object). In that case the transaction will have to fall back again to one of the other options.

If there is no pending alternate vector update or such an update completes within the time limit, then :term:`alternate selection` is performed, selecting a specific alternate for the request. This can result in a selection of "none" meaning no alternate was suitable which is always the case for a new object. A selection of "none" always results in a request to origin to add a new alternate to the alternate vector. If an existing alternate is selected then it is first checked for *coverage*. This is matching the content that is cached against the content requested and current active requests. The request is covered if all requested content is already cached. If the request is not covered then an origin request is made to retrieve the missing content. In this case, if the alternate is stale, then none of the existing content is considered valid when computing the content to request from the origin. Once |TS| is committed to an origin request it should get fresh content [#fn-uncovered-stale]_.

After the alternate is selected and verified to cover the request it is checked for freshness. If it fresh then it is served to the user agent without further action. If the content is stale the alternate is :term:`revalidated` by sending an conditional request upstream [#fn-partial-revalidate]_. This puts the object in to the pending alternate vector update state. Conceptually, at this point, the user agent request transitions to the state of having a cache hit with an alternate vector update pending. This enables the transaction to serve stale content while the revalidation happens asynchronously, or even to wait a little bit for a fast origin and then give up to serve stale.

Active requests are recorded in the ODE and content in active requests that has not yet arrived is considered when determining if a request is covered. This drives the implementation decision to separate reader and writer CacheVC instances because the content in the origin request can be very loosely related to the content in the user agent request depending on what has been cached and what other user agent requests are active. When the missing content is determined and is not empty then a request is sent to the origin to retrieve enough data to cover the missing content. Because this is a cache fill and not an alternate vector update it does not block other requests and therefore multiple cache fills can be active on the same alternate.

The point of doing coverage calculations against pending content is to reduce the load on origin servers by fetching content the fewest number of times. In some cases administrators would prefer to trade off origin load for faster responses to the user agent. The most common instance of this is video content serving. In that case if a user agent has requested the full video content and another user then requests only a range of the video it can be the range is a "long way" from the currently streaming content for the full request, forcing the other user to effectively wait until the first user gets that part of the video. This can be an unacceptably long wait (consider a 200MB video where the other user just wants to watch a it near the end). For this reason there is a configurable limit on how many bytes a request should wait to get pending content. If the wait is longer then the user agent request will create an origin request even though it duplicates content.

Previous Features
=================

There are several current features that are eliminated in the partial caching implementation. Instead these features are built more directly in to the basic mechanisms so equivalent functionality is still available.

Read While Write
----------------

There is no mention of "read while write" in the previous section because it is no longer a distinct case. Any cache operation that is not a fresh write to cache of a full request is implemented internally as read while write. Any request to the origin that will cache content gets a CacheVC as a writer. This becomes effectively independent of the user agent request. In the special case noted previously the response from the origin is split via :cpp:class:`HttpTunnel` for performance reasons but in any other case a CacheVC to read content is created. This CacheVC handles cached and live content directly, fetching from ram cache, or disk, or waiting for a writer CacheVC to fill.

For this reason the previous implementation of read while write has been completely removed, replaced in its limited scope by this more general mechanism. What was "read while write" is now the general case for all cache operations. The new implementation is more efficient and robust as well

Stale While Revalidate
----------------------

Allowing the alternate selection to select stale content enables the serving of stale content while revalidating. The implementation internally puts the stale earliest cache key in the ODE for use in this situation. Because only one alternate vector update is permitted at a time only one cache key slot is required.

Cache Write Fail Action
-----------------------

The cache object write lock is now only held during the alternate vector update pending time. If there is no request with pending response headers then any new transaction can access the cache object and serve from it or initiate an additional request if some content is missing or will take too long to return. This cannot be made better as selecting an alternate requires the alternate vector to be stable and a new request will not get the response headers needed back any faster. Even so, requests can pile up during the alternate vector update time. For this reason a timeout is available to give up on waiting for the response headers and take another action. These cover the cache write fail actions and the timeout is effectively the retry time multiplied by the number of retries. Because of how the new implementation is structured there are no longer any "retries" - the waiting CacheVC instances are placed on a list and explicitly rescheduled only when the update is complete.

Other Implementation Notes
==========================

Much effort was made to avoid any sort of retry in the cache logic. An additional link list pair was added to the CacheVC to allow instances to be placed on pending lists. When a CacheVC cannot obtain a resource because it is in use by another CacheVC it adds itself to a list to be notified when the resource is available. For example if a reader CacheVC needs content that is not cached but is scheduled to arrive due to an active origin request it adds itself to a list associated with its alternate with an internal note of which fragment in the alternate it needs. When a writer CacheVC caches a fragment, it grabs the list and walks it, scheduling CacheVC instances that were waiting for that fragment and putting the rest back on the list.

This helps with error cases as well. If a writer CacheVC fails it can easily notify any readers that the write has failed and schedule them to handle that case. The writer can even check if there are multiple writers if any other writer will deliver the content and not provide the notification in that case.

The cache data strutures are still locked but now are locked only while being actively updated so the locks are held for much shorter times. The equivalent of the cache object write lock is now to lock the ODE, update the state to indicate a alternate vector update, then drop the lock. Other CacheVC instances can then check this and arrange to wait as needed rather than failing to lock and retrying the low level lock.

Revalidation
============

Revalidation occurs only after a request has done alternate selection. That particular alternate is checked for requiring revalidation and if so the state machine performs a proxy request to the origin. It is not an object that is revalidated but always a specific alternate.

If the response is a header only update (which generally means a `304` response) or the response for whatever reason is uncachebale, then the alternate key (earliest) key remains the same and only the first doc is written to disk. If a cacheable response with content is received then the alternate is replaced and the earliest key changed. The writer CacheVC stores the original alternate key in :cpp:member:`CacheVC::update_key`. The alternate vector is not updated until the writer CacheVC is closed. At that point the earliest key for the new content is copied to the alternate vector.

Because of this the original alternate content remains available in the alternate vector until the replacement has been completely written to the cache. However this still has timing problems in that a reader CacheVC can being serving the stale content just before the final replacement content arrives and therefore still be serving from the stale data when the alternate vector is updated (changing its apparently local :cpp:member:`CacheVC::alternate`).

.. note::

   It is not yet clear when the request and response headers in the alternate vector are updated. This may happen when the origin server response header is merged with the cached response header. :cpp:func:`CacheVC::set_http_info` updates :cpp:member:`CacheVC::alternate` but shallowly so that the alternate vector is unaltered. As best I can tell the cleanup logic depends on an alternate never getting updated more than once while the :cpp:class:`OpenDirEntry` exists. If multiple writers are not allowed then the :cpp:class:`OpenDirEntry` is destoyed when the write completes which in turn destroys the newly allocated alternate. If multiple writers are allowed it's not as clear but I think it's presumed these must all be for different alternates and therefore when the last writer CacheVC closes it will trigger the same cleanup as the single writer case. There is in fact a comment implying that in :file:`!CacheWrite.cc` which states that only the first writer of multiple writers can do an update which in turns means an alternate can't be updated twice. This implies that alternates are not destroyed until the :cpp:class:`OpenDirEntry` is destroyed.

Background Fill
===============

Background fill is a configuration option that causes |TS| to continue retrieving content for an object even if the user agent disconnects. This should still work with partial caching in the same way. The utility may be much reduced because currently without background fill any retrieved content is lost if the entire object is not cached. With partial caching any full fragment that arrives will be cached, meaning that if the user agent disconnects part way through the content servered to the user agent will be cached and available for other requests even if the connection to the origin server is dropped.

However it may be the case that background fill is used in such situations as a form of pre-fetching. That doesn't work as well for range requests if the user agent requests only its desired range. For this reason it may be useful to provide a configuration option to extend range requests to fetch more content to be cached but not served to the user. If the object demographics are such that most objects are smaller than a reasonable pre-fetch size this could be set to pre-fetch such objects (effectively converting the range request into a full request to the origin) or providing some buffering room so that future full requests can overlap serving cached content and fetching new content from the origin.


.. rubric:: Footnotes

.. [#fn-uncovered-stale]

   This is a form of revalidation in that the stale content should all be discarded from the alternate and placed in the stale buffer in the ODE.

.. [#fn-partial-revalidate]

   When the alternate is stale but the request is partial, should a conditional partial request be sent to the origin? That's proably reasonable. If the response is 304 then all is good, otherwise the current content should be discarded and replaced with just the partial response. Note that this doesn't stack like a normal content fill because it is also a pending alternate vector update.
