.. include:: common.defs

.. _plugin-coordination:

Plugin Coordination
*******************

It has been requested for a long time to be able to have coordination among plugins. The two most
commonly requested capabilities are

*  Force the callback for plugin A to run before / after the callback for plugin B on a specific hook.
*  Disable a plugin and / or callback for a specific hook for a specific transaction.

I had a version of this ready for production testing, based on the Yahoo! 5.3 fork of |TS|. Based on
that work there are a number of other features that are implicit in making the primary capabilities
possible.

Modular hook dispatch.
   To make this work consistently, it is very desirable to create a class to handle the hook
   dispatching. This provides a uniform code base to handle callback dispatch.

Continuation tracking
   Each continuation needs to be tracked back to the original plugin (or core) that created it in
   order to correctly handle when or if the callback is dispatched. This has the ancillary benefit
   of being able to provide more detailed debugging and failure messages when something goes wrong
   with a continuation. In theory this may also make it possible to handle plugin reload so that
   events for continuations associated with an unloaded plugin can be dropped instead of brokenly
   dispatched.

Plugin API to probe hooks for callbacks.
   If callbacks are going to be controllable from plugins, it is also needful to allow plugins to check the callbackson a hook. This is another plugin developer request that has been around for a while.

Plugin API to manipulate callbacks on hooks
   Once callbacks are accessible via inspection from the previous point, a request for the ability
   to manipulate them is inevitable.

Phases
======

The full change is large and should be broken up into smaller updates, each of which provides some
independent value. The natural progression is basically the feature list from the previous section. In this progression each update depends on the previous one.

#. Modularize the callback dispatch. Handle the three levels of dispatch (global, session,
   transaction) in a consistent way.

#. Track continuations. Each continuation is either from the core or from a plugin. Associate the
   data for this with each continuation and whenever a continuation is created use the current plugin
   context to mark it.

#. Plugin priorities, implementation and configuration. This assigns priorities to plugins and
   dispatches callbacks in priority order. This must also include the capability to configure at
   least statically the priorities for the plugins.

#. Plugin priority API. This enables plugins to manipulate plugin priorities in code. This includes
   both adjusting priorities and enabling / disabling plugins for a transaction, and setting the
   priority threshold for a hook.

#. Callback inspection plugin API. Enable plugins to examine the callbacks associated with a hook.

#. Callback manipluation plugin API. Enable plugins to manipulate the callbacks on a hook, including
   changing priorities and enabling / disabling specific callbacks or plugins for a hook.

History
=======

At one point (Feb 2016) I had a working prototype of plugin priorities based on the YahoO! 5.3.x
fork which assigned priorities to plugin callbacks along with an API to manipulate not just the
current plugins priorities but that of other plugins. Although useful, to make it work in production
it was necessary to add plugin level enable / disable. The code for this is
`here <https://github.com/SolidWallOfCode/trafficserver/tree/plugin-coordination>`__.
