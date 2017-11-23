.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

Layer 4 Routing
***************

Layer 4 routing is the capability to handle inbound connects at the TCP or TLS level. This is
becoming a desirable capability with the increased use of TLS. While as a standalone capability it
is not particularly useful, it is becoming commonly the case that given an existing |TS| based
infrastructure, adding layer 4 routing to it is useful.

Transparency
============

Transparent connections are related to layer 4 routing. The normal transparent case isn't, because
it involves full participation by the HTTP state machine, but the blind tunnel option is in effect
layer 4 routing. The actual contents of the byte stream are not examined but are simply forwarded
based on the addresses provided by the TCP layer. While a start, this is not at all adequate for
current use cases.

SNI Routing
===========

This is a first pass at providing some useful layer 4 routing. It has several issues which need to be fixed.

*  The destination address and the URL for the ``CONNECT`` should be separately specifiable.
   Otherwise a plugin is required in order to successfully use the functionality.

*  It should be possible to specify a direct connection and not one via an HTTP ``CONNECT`` request.

With regard to the latter, the primary implementation problem is resolving an address for the
upstream destination. In the transparent case there is no need for such resolution as the address is
implicit in the inbound connection. This does not seem insurmountable, however. The main requirement
to do the resolution is a continuation to which the HostDB invocation can signal when the address is
ready. For possible existing code that accomplishes something similar we should check the
transparent bypass logic.

Separating the upstream destination and the URL for the ``CONNECT`` seems very straight forward.
