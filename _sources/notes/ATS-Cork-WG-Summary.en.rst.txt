Traffic Server Cork Working Group
*********************************

My view on the working group Cork, heavily biased toward things of importance to me.

Configuration
=============

Lua configuration has been abandoned. It will be replaced by YAML. The TsLuaConfig project will
be abandoned and the schema work converted to YAML. It was agreed that YAML based schemas will be
supported if not required to provide more accurate configuration feedback. In the longer term I
would like to build editor support logic for editing Traffic Server configurations such as for
Visual Code that can provide direct schema based feedback and error checking. It was also agreed
that we would strive for a zero configuration capability, to enable local configurations to
contain only locallly relevant configuration options.

One advantage of YAML is it lends itself to REST style API control more easily. It was the
concensus we should move forward on extending and broadening external API control of the
configuration using YAML.

Building
========

Upgrade to Clang 6 static analyzer. This created around 50 new errors which we fixed during teh
working group. There appear to still be a few issues that are problems with the analyzer we are
working around but this was determined to be overall an improvement.

The compiler requirement will be moved to C++17 starting with 8.0. The language changes are
minor, the primary advantages will be to simplify the build environment to a single version of
C++ and to gain access to many useful standard library mechanisms, such as ``std::filesystem``
and ``std::string_view``.

Process Management
==================

``traffic_cop`` has been removed for 8.0. This removes internal health checking from Traffic
Server. The concensus view is that this is done better using other mechanisms that are commonly
available now but weren't back in the early days of Traffic Server. Some additional documentation
will be required for groups wanting to use Traffic Server but not aware of specific tools needed.

It is planned to remove ``traffic_manager`` for 9.0. The latter will be significantly more
difficult because ``traffic_manager`` performs useful functions. Among these are

*  Opening of the proxy ports. This can be shifted without much loss to ``traffic_server`` which
   is already capable of doing this. This will mean lack of open ports while ``traffic_server`` is
   restarting which was considered a minor issue.

*  External communcation with command lines tools. The ongoing RPC work will be needed to move
   this functionality in to ``traffic_server``.

*  Access to statistics and metrics. This data should already be present in ``traffic_server``
   and if the RPC work is completed this should be a relatively straight forward project.

Internal RPC
The current ``libRPC`` project will be abandoned. Instead a third party RPC serialization
mechanism will be selected and used. This replacement will be required to achieve the design
goals of ``libRPC`` --

*  Separately library without executable specific compilation or libraries.

*  Bidirectional.

*  Type safety.

*  External registration to enable subsystems to register their own RPC calls and handling.

This is not as simple as it sounds. One key point of the current RPC is it requires no
allocations. Because it is a simple serialized list of fundamental types the reciever can provide
an array or argument list of those types to transfer data directly from the serialized data. It
is likley a new RPC will be YAML or (roughly equivalently) JSON based. How the memory for storing
this is to be handled will be something to be considered.

Layer 7 Routing
===============

The general direction of the work as accepted. The main concern is providing intermediate
deliverables rather than trying for one big result. The current work on the extended host
database was well accepted. In fact there was some interest in extending this to the ``HttpSM``
for other plugins. This could be an interesting idea even with regard to Layer 7 routing. There
is a problem in the design of the resolvers regarding maintaning state during resolution for a
transaction. The problem is the amount of state data depends on the resolver statck. If this data
could be added in the same way to the ``HttpSM`` then that would be a perfect spot to store that
state.

Hash Tables
===========

For concurrent containers we will try TBB (Thread Building Blocks) first. The primary constraint is
the availability of packages from well known repositories. TBB is a bit heavy weight wih many things
not of direct use but if installable via normal package management this is acceptable.

If TBB doesn't work out then we will look at either ``libcds`` (Concurrent Data Structures). A third
option is to do a fork of CK (Concurrency Kit) to make CK++, a C++ fronted version of CK.

For basic hash tables to replace the TCL hash table we will look at using the STL containers with a
custom allocator. It's still a bit unclear to me what the goal of this is. As best I can tell the
objection is that common use of the STL based containers will have memory use characteristics. A
custom allocator should be able to do better in terms of locality and cleanup. Given the strictures
on allocators I am not confident this can be done in a reasonable way.

Plugins
=======

Some plugins were promoted which will affect building but not release packages. The ``gzip``
plugin was renamed to ``compress``. This will affect packaging and configurations.

There was agreement that we should push forward on providing access to C++ based support present
in ``libtsutil``. The primary issue will be setting up the header files to avoid pulling in
internal header files which are specialized for the core. Some work was done on this at the
Working Group but more may be required.

The pre-fetch plugin written for Tumblr was discussed. The plugin itself may not be accepted in
to open source but the basic functionality was supported and it was agreed to provide it in one
way or another. It might be by extending the ``background_fetch`` plugin or some other mechanism.
There were some suggestions about pre-fetching, including using 0 or 1 byte range requests to
trigger ``background_fetch``. I'm stylistically opposed to this because it's dependent on
installing a plugin to use in a non-obvious. it might be reasonable to expand ``prefetch`` to
handle background fetch operations or vice versa. We will need to look at the configuration to
see which is best, but the shift in configuration to YAML might be a good opportunity shift thist
around.

Proxy Protocol
==============

It was agreed to commit the work on `PROXY protocol
<https://www.haproxy.com/blog/haproxy/proxy-protocol/>`__ to 8.0 with one more change. This change
is to enable PROXY protocol on a per proxy port basis using the proxy port descriptors. The keyword
for this will be ``pp``, therefore to enable PROXY protocol with TLS on port 443 the descriptor
would be ``443:ssl:pp``.

The current version of PROXY protocol is inbound only. In the longer term it will be useful to
provide for it outbound as well. I want to look at doing a betterjob of handling protocols on the
proxy ports in order to prepare for non-HTTP proxying.

Memory Allocation
=================

Comcast indicated some concerns about leaks due to disabling global allocators, which was a puzzle
both to them and to the rest of us. Nevertheless, it was agreed to proceed cautiously for this
reason. Fei's work will be to provide flags for removing just the global allocators and for removing
global and proxy allocators. In addition he will need to test this and perform measurements to
validate the changes are not harmful.

Testing
=======

The AuTest workshop went well -- everyone at the Working Group has now written an AuTest. Jason
received some useful feedback on improving the infrastructure. I discussed off line potential
additions to the AuTest tooling with regard to utilities to drive traffic through Traffic Server.
Apple is working on a tool which could be used to provide HTTP specific traffic rather than using
the more blunt tool of ``curl`` and ``grep``. In particular this would have the ability to focus on
specific headers rather than a generic text comparison, something I have wanted for a long time.

Apple also discussed their "CDN Checker" which is very similar to what I have been calling "live
testing". In essence requests are done against an in production box to verify its status and
configuration. This is the basis for the HTTP traffic tool discussed just above. In the Apple setup
this is done in an automated fashion but both I and the Traffic Control group would like to be able
to do it on demand as well.

Fine grained debugging
======================

This started from a request by Dave Carlin to be able to enable debug messages for a specific user
agent IP address. There was quite a bit of controversy about this, oddly about the configuration not
the implementation. The actual mechanism was put in 7.0 (ported from our internal patch) but 7.0 had
no way to actually use it (that was local to OATS). The concensus at the Working Group was to put
that in to 8.0 but there was some desire to have more features, such as subnets instead of a single
IP address and plugin control of the feature. There was also discussion of integration with another
debugging feature where a specific but limited set of debugging messages can be enabled on a per
transaction basis. This feature is a bit of a hack, we will need to look at how to do better. This
is a bit tricky because the per user agent IP address mechanism depends on tagging continuations
with a debug enabled flag, and by the time the :code:`HttpSM` is created many of those continuations
may already exist and not inherit the enable flag.

Other Notes
===========

Susan's design for outbound HTTP/2 was accepted. She was able to spend a good bit of time with
Kees, Masaori, and Masakazu to make sure everyone working in this area is on board with her
design.

For half closed connections, it was agreed these are not likely to be of significant use in real
production because this is not an issue for TLS or HTTP/2 connections. Susan agreed to try to
gather some actual data on how often this happens to verify.

WeAmp is preparing to release its ``fastCGI`` plugin. This enables remaps to redirect to a
fastCGI handler. This would among other enable serving static pages from Traffic Server directly
which has been a request from the operations teams and some properties for quite a while. This
could be integrated with the ``escalate`` plugin or the internal error page templates. Health
checks could use this feature. It does require an additional process to handle the fastCGI
requests which need to be managed.

Based on some work by Masoari for doing better connection draining in Traffic Server, Susan and I
are investigating doing clean cutovers from one ``traffic_server`` to another. This would involve

*  Start the new instance, wait for it to become ready.
*  Mark the old instance to drain.
*  Disable cache writing in the old instance.
*  Enable cache writing in the new instance.
*  Start accepting connections on the new instance.
*  When the old instance has few enough connections, shut it down.

The key point is to avoid having two processes write to the cache at the same time, while also
continuing to provide some level of protection to the upstreams.

AMC Tech
========

Resistance to my work on :code:`BufferWriter` finally collapsed and I will be able to merge in my
updates, which are already incorporated in to to the upstream connection limiting.

:code:`MemArena` was accepted with little comment, I will be doing a few more fixes to that before
using it in other project. The same for :code:`MemSpan` and :code:`MemVec`. I do need to track the
emerging C++20 standard on this point to minimize disruption, or provide facilities that are needed
but won't be in the future STL version.

The move to C++17 will enable me to remove some sub-optimal code, particularly in the Cache Tool.

There was a push to remove :code:`TSHashTable` in the process of doing the hash table upgrades. I
need to do some writing on that, I'm not sure it's a good idea in retrospect after pondering it for
a bit. That class does provide some features that are not going to be available in the STL
containers. I need to be specific about that so a reasonable decision can be made.
