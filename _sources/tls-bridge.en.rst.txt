.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp
.. |Name| replace:: TLS Bridge

|Name|
**********

This plugin is used to provide secured TLS tunnels for connections between a Client and a Service
via two gateway |TS| instances. By configuring the |TS| instances the level of security in the
tunnel can be easily controlled for all communications across the tunnels.

Description
===========

The tunnel is sustained by two instances of |TS|.

.. uml:: uml/TLS-Bridge-Structure.uml
   :align: center

The ingress |TS| accepts a connection from the Client. This connection gets intercepted by the
|Name| plugin inside |TS|. The plugin then makes a TLS connection to the peer |TS| using the
configured level of security. The original request from the Client to the ingress |TS| is then sent
to the peer |TS| to create a connection from the peer |TS| to the Service. After this the
Client has a virtual circut to the Service and can use any TCP based communication (including TLS).
Effectively the plugin causes the connectivity to work as if the Client had done the ``CONNECT``
directly to the peer |TS|. Note this means the DNS lookup for the Service is done by the peer |TS|,
not the ingress |TS|.

The plugin is configured with a mapping of Service names to peer |TS| instances. The Service
names are URLs which will in the original HTTP request made by the Client after connecting to the
ingress |TS|. This means the FQDN for the Service is not resolved in the environment of the peer
|TS| and not the ingress |TS|.

Implementation
==============

The |Name| plugin uses :code:`TSHttpTxnIntercept` to gain control of the ingress Client session.
If the session is valid then a separate connection to the peer |TS| is created using
:code:`TSHttpConnect`.

After the ingress |TS| connects to the peer |TS| it sends a duplicate of the Client ``CONNECT``
request. This is processed by the peer |TS| to connect on to the Service. After this both |TS|
instances then tunnel data between the Client and the Service, in effect becoming a transparent
tunnel.

The overall exchange looks like the following:

.. uml:: uml/TLS-Bridge-Sequence.uml
   :align: center

A detailed view of the plugin operation.

.. uml:: uml/TLS-Bridge-Plugin.uml
   :align: center

A sequence diagram focusing on the request / response data flow. There is a :code:`NetVConn` for the
connection to the Peer |TS| which is omitted for clarity.

*  Blue dotted lines are request or response data
*  Green lines are network connections.
*  Red lines are programmatic interactions.
*  Black lines are hook call backs.

The :code:`200 OK` sent from the Peer |TS| is parsed and consumed by the plugin. An non-:code:`200` response
means there was an error and the tunnel is shut down. To deal with the Client response clean up the
response code is stored and used later during cleanup.

.. uml:: uml/TLS-Bridge-Messages.uml
   :align: center
