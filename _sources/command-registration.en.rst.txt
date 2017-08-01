.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _command registration:

Command Registration
********************

Command registration is registering *tags* that correspond to actions for a command line tool to execute. There is a tree of tags so that more specific commands are done by listing multiple tags. E.g. "list" and "list stripes" are different actions and each tag controls which tags are allowed to follow. For exanmple :program:`traffic_ctl` with commands like::

   traffic_ctl shutdown
   traffic_ctl metric set proxy.config.http.cache.enable 0

Internally this is done in :program:`traffic_ctl` in an ad hoc fashion where each action is done by
invoking a function and each such function must handle dispatching to any subcommand tags. This
class is an attempt to make that generic so that action functions are leafs of command line parsing.
This reduces cut and paste code along with making the tag hieararchy controlled from the top level
of the program rather than being scattered across the implementation.

