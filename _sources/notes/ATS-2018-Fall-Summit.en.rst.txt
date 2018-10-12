2018 Fall Summit
****************

These are my notes from the 2018 Fall ATS Summit.

Reloadable Remap
================

Gancho, Zeyuan, Aaron and I had some talks about Gancho's project for reloadable remap plugins.
Gancho's work deals with the continuation tracking by providing the plugin handle explicitly
to the initialization logic and then adding new API calls to created tracked continuations. This is
the same technology the Plugin Coordination project requires.

I think we're on the same development page now. Rather than changing the API to accomodate the
plugin information / handle, Gancho will use the same thread storage context technique I had planned
for Zeyuan's plugin coordination work. This will make eventual convergence easier, although for now
Gancho will work on just the remap plugin information and see how that works and then we'll work on
unifying the global plugin and remap plugin information structures.

Gancho's primary work is on how to reload an already loaded ``.so`` that has been changed on disk,
and tracking the number of continuations for a ``.so`` so the reload can be delayed until there are
no more continuations that can be dispatched.

It will be necessary to look at some of the other hook dispatching and either adjust it to track or
to build more generic support for dispatching callbacks that are not state machine related (e.g.
lifecycle or TLS hooks).

YAML
====

I talked with Randall about where we want to go with YAML configuration and we're in basic agreement.

*  Begin with smaller projects to convert to YAML, work up to the more complex configurations.

*  Plan for convergence - independent YAML configurations should be designed to be consolidated
   into larger configurations, with the ultimate goal of a single configuration file.

*  Use schemas for validation, error reporting, and editor support. These will be JSON Schema based.

I described my work on Canned-YAML and Randall thought that would be useful. Bryan and Leif thought
I should plan for it being independently open sourced. This would be easy since it's already in a
separate repository.

Issues
------

An open issue with YAML is how to provide the library to plugins. We definitely want plugins to use
YAML as much as possible. Making the core version of libYAML-CPP available will be critical to
making this happen.

Additionally, if the configuration files are done in a way that makes convergence possible, it may
be reasonable to take more of a "config.d" approach and simply load all files with a specific suffix
from a directory into one large configuration. This is a common style, and it obviates the need for
any "include" functionality. In effect all files in the configuration directory are included into
a constructed configuration.

For the IP Allow YAMLization, I kept the old parsing logic so that for ATS 9 either format would
work, on the presumption the old format would be deprecated in ATS 9 and dropped in ATS 10. Whether
this should be a general pattern is not yet decided.

remap
-----

There was general agreement we should take the opportunity to redesign from scratch to take
advantage of YAML and provide new features that have been requested for a long time. Randall agreed
to take point on organizing this effort.

An open issue is how to mark value types. The archetypical example is host matching for remap. It
is important to be able to specify both literal host names and regular expressions for host names.
How is the configuration marked to distinguish these cases?

#. Use the YAML type tagging to mark regular expressions as ``!!regex``.

#. Use a second boolean value which is set if the primary value is a regular expression.

#. Use a distinct name for literal vs. regular expression values, e.g. "host" vs. "host_rx".

Plugin Coordination
===================

There was no support for static configuration of plugin ordering, therefore that phase will be
changed to guarantee the ordering in ``plugin.config`` is the ordering of callback dispatch and
document that.

No one had a use case for thresholds either, so that phase will be dropped.

Otherwise it was well received and the remaining work will proceed.

Thread Affinity
===============

Good discussions were done for the log rotation issue (where small rotated logs get swamped by large
ones) and thread affinity. The concensus was

* :code:`INKContInternal` instances would get a default thread affinity of the thread on which they
  were created.

*  Is it not an error to schedule a continuation on a thread pool to which it is not affine.

*  If a continuation is scheduled on a pool and it is affine to a thread in that pool, it will be
   scheduled on the affine thread. This isn't a real functional change as scheduling on a pool
   allows the core to select any thread in that pool.

ATS 9
=======

ATS 9 is shaping up to be focused on plugins and configuration. A good number of new plugins have already been contributed. There are also several infrastructure projects underway to better suppport plugins.

*  Moving core utilities to where they are accessible by plugins.

*  Making remap plugins completely reloadable.

*  Plugin coordination, ordering callbacks on hooks and providing plugin introspection.

*  Thread affinity and control for plugin continuations.

For configuration, we expect all of the configuration files to be redone as YAML by the time of the
ATS 9 release.

Miscellaneous
=============

Daniel Morilha agreed to work on doing Proxy Protocol outbound.

Leif said we need to have more hacking sessions, to fix bugs, discuss issues, and review pull
requests.

For traffic replay, the basic engine will be exported as an independent open source project. There
was good interest in this functionality. Leif wanted to do better performance testing and in
particular try performance tests on previous version to get a better baseline. This is a good idea,
the only impediment being finding someone who has the time to do it.

Log file culling will be changed. A "min-count" configuration will be added to specify the minimum number of rolled log files to keep for each log file type. When disk headroom is below the minimum, rotated log files will be deleted as the oldest log file of the type with the largest ratio of rolled files to minimum file count. This, among other effects, will mean that deletion of a rolled log file will always pick a type that has more that it's minimum count before any other log file type that is at or below its minimum count. The solves the original problem, which was protecting smaller log files from get clipped at the same age as large log files.

