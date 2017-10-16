.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _external types:

External Types
**************

Traffic Server
==============

.. c:type:: TSReturnCode
.. c:type:: TSHttpTxn
.. c:type:: TSIOBufferReader
.. c:type:: TSIOBufferBlock
.. c:type:: TSIOBuffer
.. c:function:: void * TSmalloc(size_t n)
.. c:function:: char * TSstrdup(char * str)

.. type:: ts
.. namespace:: ts
.. class:: string_view

   A local implementation of the C++17 `std::string_view <http://en.cppreference.com/w/cpp/string/basic_string_view>`_. See :ts:git:`lib/ts/string_view.h`.

.. class:: Errata
.. namespace:: NULL


STL types
=========

.. type:: std
.. namespace:: std
.. type:: string
.. type:: template <typename T> function
.. namespace:: NULL

Builtin Types
=============

.. type:: size_t
.. c:type:: int64_t
.. c:type:: bool

.. _custom-logging-fields:

Reference Stubs
===============
