.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _rpc_refactor:

Administrative JSON RPC
***********************

To communicate between ``traffic_manager`` and ``traffic_server`` there is an RPC mechanism in
``mgmt/api``. This is a simple serialization style RPC which runs over sockets. The goal of this
project is to replace this with a `JSON-RPC <https://www.jsonrpc.org/specification>`__ based implementation. The key benefits would be

*  A standard RPC mechanism that is widely supported.
*  Enable internal extensibility.
*  More robust argument handling.
*  Synchronous management actions (also, remove unnecessary delays).
*  Zero config, dynamic configuration.

The current logic is hand rolled and the messages and arguments are deeply embedded in the implementation, which makes changing them difficult and risky. This has an effect on most of these properties. A clean implementation that is generic would do much all by itself to ameloriate most of these issues, irrespective of it being JSON-RPC.

Standard RPC
============

Currently only |TS| specific implementations can access the external management API. This means interacting with any sort of standard managemet tools is very difficult, requiring custom code. |TS| ships with a Perl library to enable access via Perl, but this code is old, unmaintained, and painful to build.

In contrast, a JSON-RPC based API opens up a wide variety of libraries and tools in most (if not all) of the popular languages. There would be no need to provide any explicit support in the |TS| code base, as these existing tools are better than anything our community is likely to build.

One concern is performance, but in general the performance contraints for management APIs are much less severe than production traffic and in my view not a serious blocker for this application.

Internal extensibility
======================

The functionality of the management API is limited primarily because it is so difficult. The new implementation must support a subsystem adding message types to the RPC without code changes in the RPC library. This will make adding additional functionality easy and make providing additional features much more feasible.

Argument handling
=================

This is a mixed bag. On the one hand, using JSON-RPC will mean sending message arguments as JSON encoded data, effectively strings. This means converting to and from strings as the messages pass. Although it is a judgement call to say that will not be a problem. one can point to the large number of production web servers that use JSON-RPC (or just JSON encoded data) at production traffic levels.

On the other hand JSON encoded data is very easy to pass around and extend. If a new management API function needs a new type, it does not need any modification or additional support in the RPC library. The RPC library will be expected to handle only JSON encoded data. It may the case that an encoding framework is provided, but in general I have found it easier to directly print JSON encoded data than to use such a framework. Decoding is somewhat more challenging and we will need to look at what support can be provided by the RPC library. The RPC library will be responsible for parsing from text to JSON (or equivalent) data strutures. Other core components should need only to do direct type conversions. The RPC library must provide a set of basic converters for standard JSON types (integer, floats, booleans, possibly a few others).

Synchronicity
=============

A key problem from the operators point of view is the "fire and forget" style of the current management API. The command line tools can verify the *reciept* of a message, but not any action done because of that message. The new implementation must be able to support synchronous messages for two reasons.

*  First so that the operators (or their tools) can know the message has been acted upon.

*  Enable message processing to pass back diagnostic information directly to the caller, rather than by spewing to a log file in the (frequently vain) hope an operator will notice.

It is acceptable for synchronicity to depend on the message, or on flags in the messages, but it must be possible.

In addition, there are multi-second delays built in to the current management API, for no particularly good reason I can see. A polling style is used, which seems unnecessary. Eliminating those delays would be worth while by itself, independent of using JSON-RPC.

Dynamic configuration
=====================

I dare to dream big - I want a |TS| which can start up with almost no (or literally no) configuration and then be fully configured via the management API. It has been a goal to minimize the required configuration need for a working |TS| for other reasons, but as the ability to configure via a management API is, in my opinion, critical for the long term success of |TS|.

As cloud based computing increases, operators will want ot embed |TS| in such systems (e.g. inside Kubernetes). Having to package configuration files is painful in such environments. What is wanted is the ability to deploy with a small amount of generic configuration (e.g. something that can be embeded in a RPM package) and bootstrap from that to the specific configuration needed.

Code Structure
==============

As mentioned, the new implementation should be done in a library style, so that it depends on (at most) the already existing "tscore" library, but *none* of the proxy specific code. This is, in my view, the greatest single issue with the current code, its excessively complicated dependencies and linking. The new implementation must be a simple library with simple dependencies that don't reach all over the code base.

Although not needed in the first phase, it should be kept in mind that the ability of *plugins* to register management API calls would be a very powerful feature. The design and implementation of this first phase should keep this in mind as a future feature.