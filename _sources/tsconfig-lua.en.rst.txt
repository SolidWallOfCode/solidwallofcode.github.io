.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _tsconfig lua:

TSConfig / Lua
**************

|TS| has committed to moving configuration files to be Lua based. While conversion of existing files
to Lua isn't required, the current view based on mailing list discussions is that new configuration
files are required to be in Lua. This project is about providing that technology in a generic way to
make it easier to add configuration for new features.

Current Practice
================

The primary example current is the custom logging configuration file :file:`logging.config` which is
processed as a Lua file. The interface to the |TS| core is done by creating Lua types in the core
and exposing them to the Lua interpretor (see :ts:git:`proxy/logging/LogBindings.cc`). Data is
transfered by the Lua interpretor executing calls to the extended Lua types and then invoking the
|TS| core. The provided instances from the core are invoked with arguments passed by the
interpretor, which are then stored in logging configuration data structures. This works but the
downside that every subsystem that needs a configuration file will need to write specialized Lua
types to get its data.

Design
======

An alternative is to provide a more generic facility that has the Lua file construct a forest of
data and then copy that forest over to a C++ container. In order to provide appropriate feedback for
configurations a description of expected values is needed, in the flavor of the current
``RecordsConfig.cc``. Because the data is more complex this takes the form of a schema to describe
the data. The schema is a data object written in Lua, modeled on the standard `JSON schema
<http://json-schema.org>`. [#fn-schema]_

The schema drives the rest of the implementation. A C++ structure to hold the configuration values
is generated which contains both the configuration and schema based metadata. This makes schema
information available for validation and error reporting.

To use this configuration a schema is written to describe the expected configuration data. During
build time a C++ container, a "configuration class", is generated from the schema. At run time an
instance of the configuation class is created and the passed the path to the Lua configuration file.
The file is read and the data loaded into the configuration class instance. Any errors of the
configuration data is returned for logging. The calling code needs to do very little validation
because most or all of that is done by the configuration loading, based on the schema.

Implementation
==============

The schema is processed to generate the configuration class. Its structure parallels that of the
configuration data, as descibed by the schema. :class:`TsConfigBase` is the abstract base class for
the configuration class. it provides a base uniform interface for operating on elements of the
configuration class. Each configuration data element in the base configuration class it also a
subclass of :class:`TsConfigBase` and as the client calls the :func:`TsConfigBase::loader` method
to load the overall configuration, that class calls the same method on each of its elements, and so
recursively on down until primitives are reached which load themselves.

This style requires that each element have access to data from the schema for validation and error
reporting. To avoid the expense of constructing this data repeatedly it is split off into classes
based on :class:`TsConfigDescriptor`. Instances of subclasses of this are intended to be declared
staticly and the elements of the configuration class are initialized with references to these static
instances. The :class:`TsConfigBase` class hierarchy is for dynamic, per configuration data and the
:class:`TsConfigDescriptor` class hierarchy is for static schema data.

All of this is description and operations. The actual configuration data is stored directly as
members of the configuration class for easier access. Consider a schema that is a single integer.

.. code-block:: lua

   {
      cname="CounterConfig",
      global="ItemCounter",
      name="counter",
      type="integer".
      description="Number of items to track."
   }

This creates a container class.

.. code-block:: cpp

   class CounterConfig : public TsConfigBase {
      CounterConfig() : TsConfigbase{_DESCRIPTOR},
                        _meta_counter{counter, COUNTER_DESCRIPTOR}
      {}

      /// Load from the file at @a path.
      ts::Errata load(std::string const& path);

      /// Data loader method.
      ts::Errata loader(lua_State * s) override;

      /// Number of items to track.
      int counter;

      /// Descriptor for schema / CounterConfig
      static TsConfigSchemaDescriptor _DESCRIPTOR;
      /// Static schema data for @a counter.
      static TsConfigDescriptor COUNTER_DESCRIPTOR;
      /// Dynamic data for @a counter.
      TsConfigInt _meta_counter;
   }

The schema static data is defined as

.. code-block:: cpp

   static TsConfigSchemaDescriptor CounterConfig::_DESCRIPTOR {
      TsConfigDescriptor::SCHEMA,
      "CounterConfig",
      "",
      ""
   };

   static TsConfigDescriptor CounterConfig::COUNT_DESCRIPTOR {
      TsConfigDescriptor::INT,
      "integer",
      "count",
      ""
   };

Loading the configuration is done by

.. code-block:: cpp

   CounterConfig config;
   config->load("counter.config");
   // ...
   tracker.resize(config.counter); // Use loaded value.

A valid configuration example

.. code-block:: lua

   ItemCounter = 17;

Enumerations
------------

Being able to contrain primitive values to a specific set of values is critical to providing good
feedback for configurations. Data for an enumeration is stored in a static instance of
:class:`TsConfigEnumDescriptor`. This contains two mappings, from string keys to numeric values and
from numeric values to string keys. These values should be available in C++ and in Lua.

For Lua enumerations are stored as tables attached to predefined global variables. The schema
supports defining such enumerations and specifying the global variable. Note this can be a nested
path so that a schema could have a single top level global for its enumerations with each
enumeration an entry in the global table. The presumption is the enumeration values are integers and
these are the values passed to the configuration loader, specified by the global value. This is
intended to minimize undetected typographic errors in the configuration file.

Enumerations are defined in a top level key of the schema named ``definitions``. Enumerations are
restricted to mappings between strings ("keys") and integers ("values"). The mapping can be defined
in multiple ways.

Lua array of strings
   The keys are the strings in the array and the corresponding values the index of the string in the array.

Lua table
   This must be a map between strings and integers, in either direction. A mapping from integers to
   strings is flipped to its dual. Duplicated keys or values are an error.

The schema data for an enumeration is stored in a static instance of
:class:`TsConfigEnumDescriptor`. The internal maps are initialized by the constructor.

Interface
=========

.. class:: TsConfigBase

   .. function:: TSConfigBase()

      Default constructor. The configuration is populated with defaults, if specified.

   .. function:: ts::Errata loader(std::string const& path)

      Load data from the Lua config specified by :arg:`path`.

   .. enum:: Source

      Which type of source of configuration data.

      .. c:macro:: NONE

         No explicit source, data is default constructed.

      .. c:macro:: SCHEMA

         Data was taken from defaults in the schema.

      .. c:macro:: CONFIG

         Data was read from the configuration file. This value has priority - if the source is
         :c:macro:`CONFIG` for a non-primitive that means at least one nested primitive value was read
         from the configuration file.

   .. member:: Source source

      The source for the configuration data associated with this instance.

   .. member:: TsConfigDescriptor const & descriptor

      Static schema data for the configuration value.

.. class:: TsConfigDescriptor

   Base class for schema element descriptions.

.. class:: TsConfigEnumDescriptor : public TsConfigDescriptor

   Schema data for an enumeration type.

   .. function:: ts::string_view operator [] (int value) const

      Find the key for the :arg:`value`. An :code:`std::out_of_range` exception is thrown if :arg:`value` is not valid.

   .. function:: int operator [] (ts::string_view key) const

      Find the value for the :arg:`key`. An :code:`std:out_of_range` exception is thrown if :arg:`key` is not valid.

   .. function:: bool is_valid(int value) const

      Check if :arg:`value` is valid.

   .. function:: bool is_valid(ts::string_view key) const

      Check if :arg:`key` is vald.

Schema
======

A Lua configuration schema has its own schema.

.. literalinclude:: lua-config-meta-schema.lua
   :language: lua

Example
-------

An example project using TsLuaConfig is SNI remap. An example configuration looks like

.. code-block:: lua

   sni_config = {
      { fqdn="one.com", action=TLS.ACTION.TUNNEL, upstream_cert_verification:TLS.VERIFY.REQUIRED}
   }

The schema to describe this configuration is

.. literalinclude:: lua-config-sni-remap.lua
   :language: lua

This generates a config class.

.. code-block:: cpp

   struct LuaSNIConfig : public TsConfigBase {
      using self = LuaSNIConfig;
      enum class Action { CLOSE, TUNNEL };

      static TsConfigArrayDescriptor DESCRIPTOR;
      LuaSNIConfig() : TsConfigBase(DESCRIPTOR), DESCRIPTOR(Item::DESCRIPTOR) {}

      struct Item : public TsConfigBase {
         Item() : TsConfigBase(DESCRIPTOR),
            FQDN_CONFIG(FQDN_DESCRIPTOR, &self::fqdn),
            LEVEL_CONFIG(LEVEL_DESCRIPTOR, &self::level),
            ACTION_CONFIG(ACTION_DESCRIPTOR, &self::action)
            { }
         Errata loader(lua_State* s) override;

         std::string fqdn;
         int level;
         Action action;

         // These need to be initialized statically.
         static TsConfigObjectDescriptor DESCRIPTOR;
         static TsConfigBaseDescriptor FQDN_DESCRIPTOR;
         static TsConfigString<Item> FQDN_CONFIG;
         static TsConfigBaseDescriptor LEVEL_DESCRIPTOR;
         static TsConfigInt<Item> LEVEL_CONFIG;
         static TsConfigEnumDescriptor ACTION_DESCRIPTOR;
         static TsConfigEnum<Item, ACTION> ACTION_CONFIG;
      };
      std::vector<Item> items;
      ts::Errata loader(lua_State* s) override;
   }

Future Work
===========

In the long run it would desirable to have a single Lua configuration file for all configurations.

There is a difficult question on how ``traffic_ctl`` interacts with Lua based configuration. That is
not specific to this configuration system but is a more general problem.

Some consideration must be given to whether variants should be supported. This would allow a
configuration value to be listed as multiple types and any of the types would be acceptable. It's
unclear how useful this would be. The most common case I can think of is enabling singletons vs.
arrays, but it would be easier to just do that directly in the validation stage. That is, if a type
is described as "array of T" and the value is a single T, convert it to an array of size 1. This
seems obvious and non confusing and I'm not sure there are any other good use cases.

History
=======

Originally this work was intended to be based on the existing TsConfig library. This was eventually
abandoned due to concerns about configuration error feedback. TsConfig provides detailed reporting
for errors but can do that only because it also does the parsing. With the loss of control text
parsing providing useful feedback becomes effectively impossible. Related to this was the ability to
detect "excess" configuration that didn't match what was expected. This is most commonly the result
of typographic errors and the operator should be informed of this rather the configuration silently
using defaults. This is something possible with the current system - it can at least warn of ignored
configuration variables. Once there is a mechanismm for describing which values are expected there
are many things that can be added to it at low marginal cost. The eventual result is the schema
system described here.

.. rubric:: Footnotes

.. [#fn-schema]

   While some consideration was given to using the JSON schema directly, over all it
   was considered overall better to use Lua for consistency so all of the configuration related data is
   in the same language.
