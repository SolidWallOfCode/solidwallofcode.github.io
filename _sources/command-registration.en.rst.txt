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

Implementation
**************

.. class:: CommandTable

   The root of a set of registered commands.

   .. function:: CommandTable()

      Construct an empty command table.

   .. type:: Action = std::function<ts::Errata (int, char * [])>

      A function that performs the action for a command. This action is always a leaf, any
      additional arguments available are passeed to the function rather than checked for additional
      commands. The first argument to the function will be the number of arguments and the second an
      array of constant character pointers that are the arguments.

   .. type:: NullaryAction = std::function<ts::Errata()>

      A function that performs an action for a command. This action can be used on a non-terminal
      command but can take no arguments. It will be invoked only if no more arguments remain after
      processing the tag for the command.

   .. class:: Command

         An command action.

         .. function:: subCommand(std::string const & name, std::string const & help)

            Add a subcommand to this command. The tag used is :arg:`name` along with a :arg:`help` string for diagnostics.

         .. function:: subCommand(std::string const & name, std::string const & help, Action f)

            Add a subcommand to this command. The tag will be :arg:`name`. Help for this command
            will be :arg:`help`. If the command is invoked the function :arg:`f` will be called.
            This will be a leaf tag meaning that if this tag is reached no more subcommands will be
            accessible and additional arguments will be passed to :arg:`f`.

         .. function:: subCommand(std::string const & name, std::string const & help, NullaryAction f)

            Add a subcommand to this command. The tag will be :arg:`name`. Help for this command
            will be :arg:`help`. If the command is invoked the function :arg:`f` will be called.
            This command will only be invoked if no more arguments are available, otherwise the
            additional arguments will be treated as subcommand tags.
