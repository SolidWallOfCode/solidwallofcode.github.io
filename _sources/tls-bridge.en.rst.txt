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

.. image:: images/TLS-Bridge-Structure.png
   :align: center

The inbound |TS| accepts a connection from the Client. This connection gets intercepted by the
|Name| plugin inside |TS|. The plugin then makes a TLS connection to the outbound |TS| using the
configured level of security. The original request from the Client to the inbound |TS| is then sent
to the outbound |TS| to create a connection from the outbound |TS| to the Service. After this the
Client has a virtual circut to the Service and can use any TCP based communication (including TLS).

The plugin is configured with a mapping of Service names to outbound |TS| instances. The Service
names are URLs which will in the original HTTP request made by the Client after connecting to the
inbound |TS|. This means the FQDN for the Service is not resolved in the environment of the outbound
|TS| and not the inbound |TS|.

Implementation
==============

The |Name| plugin uses :code:`TSHttpTxnIntercept` to gain control of the inbound client session.
If the session is valid then a separate connection to the outbound |TS| is created using
:code:`TSHttpConnect`.

After the inbound |TS| connects to the outound |TS| it sends a duplicate of the client request. This
is processe by the outbound |TS| to connect on to the target service. After this both |TS| instances
then tunnel data between the client and the service, in effect becoming a transparent tunnel.

For the particular use case currently planned the client will send a ``CONNECT`` which will then be
sent to the outbound |TS|. This will result in blind tunnels in both the inbound and outbound |TS|
instances giving the client a byte level secured connection to the service. Note there is no plugin
in the outbound |TS| - it only needs to be configured appropriately.

.. image:: images/TLS-Bridge-Sequence.png
   :align: center
