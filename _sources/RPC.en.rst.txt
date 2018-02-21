.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _rpc_refactor:

RPC Refactor
*************

To communicate between ``traffic_manager`` and ``traffic_server`` there is an RPC mechanism in
``mgmt/api``. This is a simple serialization style RPC which runs over sockets. Current the logic is
tightly embedded in the management library. It would be useful for various reasons to extract this
in to a separate library.

In addition there seems to be a delay between a command line tool such as ``traffic_ctl`` sending
data and ``traffic_server`` receiving the data. This is likely due to :code:`Continuation`
scheduling that causes the socket to be polled every few seconds.

The phases of the project

#. Determine if there is a propagation delay.

   a. If there is a delay, remove it.

#. Find or create documentation for the current RPC mechanism, including

   a. The basics of the serialization mechanism.

   #. The API for both sides of the RPC.

   #. The run time structure of the RPC from ``traffic_ctl`` to ``traffic_server``.

   #. A sequence diagram of how a message is delivered.

#. Factor out the serialization logic.

   a. Possibly remove the assymetry between ``requests`` and ``responses``. I think this may be the
      root of the assymmetry. See :ts:git:`mgmt/api/NetworkMessage.cc`.

   #. If this is made in to a library, it will be necessary to make the registration of message and
      type handling a push mechanism rather than a push. That is, currently this information is
      directly embedded in the RPC code, which won't be possible in a library.

   #. Make it possible for the core to pass messages back from ``traffic_server`` to command line tools.

   #. Make it possible for plugins to pass messages back to command line tools.

#. Create ``libatsprc``.

   a. Create the library.

   #. Change the RPC using code to use the library.

.. rubric:: Notes

Variadic Template Support
   I considered this for serialization but the problem is deserializing. Given a message type, how
   is a call using variadics to be constructed? You would still need some sort of registration for a
   functor to invoke, at which point it's not really winning. There would also be the problem of
   matching serialized type data with C++ types, again obviating the advantages.
