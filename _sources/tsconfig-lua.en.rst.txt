.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _tsconfig lua:

TSConfig / Lua
**************

|TS| has committed to moving configuration files to be LUA based. While conversion of existing files
to LUA isn't required, the current view based on mailing list discussions is that new configuration
files are required to be in LUA. This project is about providing that technology in a generic way to
make it easier to add configuration for new features.

Current Practice
================

The primary example current is the custom logging configuration file :file:`logging.config` which is
processed as a LUA file. The interface to the |TS| core is done by creating LUA types in the core
and exposing them to the LUA interpretor (see :ts:git:`proxy/logging/LogBindings.cc`). Data is
transfered by the LUA interpretor parsing calls to the extended LUA types and then invoking the core
provided instances with arguments passed by the interpretor, which are then stored in logging
configuration data structures. This works but with the downside that every subsystem that needs a
configuration file will need to write specialized LUA types to get its data.

Design
======

An alternative is to provide a more generic facility that has the LUA file construct a forest of
data and then copy that forest over to a C++ container. There is already an existing library of this
type, TSConfig. TSConfig had its own input file format based on `libconfig
<https://github.com/hyperrealm/libconfig>`_ plus a C++ tree container interface supporting a small
set of basic types. This interface suffices for the require LUA config interface. The approach here
is to keep the TSConfig C++ data access interface and replace the configuration file parsing with
LUA parsing and extraction of the LUA data.

The usage would be to create an instance of TSConfig, passing it a configuration file and a set of
"roots". The configuration file would be loaded and interpreted as a LUA file. After that a set of
global variables, identified by the roots passed in, would be copied in to the C++ container. The
roots could each contain a tree of arbitrary depth, the group forming a forest. This forest could
then be walked or examined in way very similar to how the current configuration data is accessed,
but with more generality.

*  Relative paths could be used rather than absolute paths.
*  Run time type checking of values.

Implementation
==============

TSConfig needs to be able to extract trees of arbitrary depth from LUA. Some work has been done with
this but there is not yet a full implementation.

On the TSConfig side, the library is currently working and should be relatively easy to adapt to LUA
because this will be mostly removing parsing code. A side effect of this will be to finally put an
end to the FLEX/Bison issues where these files get updated inappropriately causing source control
issues.

Interface
=========

.. class:: TSConfig

   .. function:: TSConfig()

      Default constructor, creates an empty configuration.

   .. function:: TSConfig& add_root(string_view name)

      Add a root. This will become available as a name in the root map of the configuration and the
      value of the global variable of the same name in the LUA state will be copied to that name.

   .. function:: bool load(string_view path)

      Load a configuration file. The file will be loaded and then interpreted as a LUA script. The
      set of global variables defined by the roots of the configuration will have their values
      copied from the LUA state to the corresponding names in the root table.

   .. function:: TSConfigValue root()

      Return the root value of the configuration. This will be a :code:`table`. The keys for the
      table will be the roots added to the configuration via :func:`TSConfig::add_root`. The values
      for the keys will be the values of the corresponding global variables in the LUA state.

.. class:: TSConfigValue

   Configuration data in a :class:`TSConfig` instance. This is a generic type that covers all of the
   possible types in a configuration. A value is either a *primitive* which is a single basic value
   or a *table* which is a container for other values. A :code:`table` can have indexed values and also
   named values, to be similar to a LUA table.

   Primitives
      :code:`nil`
         No value.

      :code:`integer`
         A signed integer.

      :code:`string`
         A string (sequence of characters).

      :code:`float`
         A floating point number.

      :code:`ip`
         An IP address.

   .. function:: TSConfigValue()

      Default constructor, creates a :code:`nil` value.

   .. function:: bool is_primitive() const

      Returns :code:`true` if this is a :code:`primitive`, :code:`false` otherwise.

   .. function:: bool is_table() const

      Returns :code:`true` if this is a :code:`table`, :code:`false` otherwise.

   .. function:: TSConfigValue operator [] (int idx)

      Get the value at :arg:`idx`. If this is not a :code:`table` or there is no value at :arg:`idx`
      a :code:`nil` value is returned.

   .. function:: TSConfigValue operator [] (string_view name)

      Get the value for the key :arg:`name`. If this is not a :code:`table` or there the key
      :arg:`name` is not in the table, a :code:`nil` value is returned.

