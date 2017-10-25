.. include:: common.defs
.. default-domain:: cpp

.. _vconn_args:

TSVConn Arguments
*****************

To enable better Layer 4 routing it would be very helpful to be extend the transaction and session
rguments to virtual connections. Plugins can then easily pass data from early intervention (layer 4)
callbacks to transaction (layer 7) callbacks. For plugins such arguments are only needed to be
attached to :code:`INKVConnInternal` and :code:`NetVConnection` as there aren't any other virtual
connection types accessible from the plugin API.

In terms of implementation the primary issue is adding a hook that a plugin can use to clean up
resources. There are already sufficient early intervention hooks to set the data but both
:code:`TS_HTTP_TXN_CLOSE` and :code:`TS_HTTP_SSN_CLOSE` hooks are insufficient in case of error
before the client session is constructed. A new hook, :code:`TS_VCONN_CLOSE`, is needed to do this
cleanup. We should rename :code:`TS_VCONN_PRE_ACCEPT_HOOK` to :code:`TS_VCONN_START`.

In terms of implementation we would add a :code:`std::vector` to a class which would be inherited by
:class:`INKVConnInternal` and :class:`NetVConnection`. This has a size of 24 bytes default
constructed. When needed the vector would be extended to a compile time constant number of entries.
This is a trade off from a constant sized array in that it is expected most instances will not have
the arguments in which no memory is allocated. Only instances that need the data will allocate,
reducing the overall memory footprint.

A sample man page is :ref:`here <TSVConnArgs>`.

The original discussion is `here <https://github.com/apache/trafficserver/issues/2388>`__.
