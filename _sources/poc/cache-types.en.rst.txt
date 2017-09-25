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

.. _cache_types:

Cache Types
*********************

.. cpp:type:: HttpCacheKey

   The type used to hold a full cache key.

.. cpp:class:: HTTPCacheAlt

   Description of an alternate in the cache. This contains the metadata for the alternate, such as the fragment table and the request and response headers.

.. cpp:class:: CacheHTTPInfo

   A wrapper class for a pointer to a :cpp:class:`HTTPCacheAlt`.

Traffic Server Types
********************

.. cpp:class:: HttpSM

   The HTTP transaction state machine, which also represents the transaction.

.. cpp:class:: HttpTunnel

   Contains the data sources and sinks for the :cpp:class:`HttpSM`.

