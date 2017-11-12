.. include:: common.defs

.. default-domain:: c

Body Factory Refactoring
************************

The core logic used to generate proxy replies is convoluted and fragile. It has needed to be cleaned
up for a long time both to make it more robust and maintainable but also to add features that are
currently hard to do. There are also arbitrary internal limits which are entirely artifacts of the
implementation.

The goal is to replace all of this with :code:`MIOBuffer` to get the following benefits:

*  Remove arbitrary limits.
*  Remove all direct memory allocation, use only :code:`IOBufferBlock` based allocations.
*  Reduce costs of using the generated bodies later. An :code:`MIOBuffer` can be directly fed to an
   output tunnel or even a socket without additional copies or allocations.
*  Simplify the code. All output is done directly to an :code:`MIOBuffer`.

The basic plan is to extend the :code:`BufferWriter` class to have one that takes an
:code:`MIOBuffer` as its output target. This would make it basically equivalent to C++ I/O stream operators
but using |TS| core mechanisms.

Implementation
==============

The starting change is

#. change :code:`HttpTransact::State::internal_msg_buffer` from a :code:`char*` to :`MIOBuffer`
#. remove :code:`HttpTransact::State::internal_msg_size`
#. remove :code:`HttpTransact::State::internal_msg_size_fast_allocator_size`
#. change :code:`HttpTransact::State::internal_msg_buffer_type` to :code:`ats_scoped_str`. This is
   used to populate the ``Content-Type`` header and so doesn't need to be a variable sized buffer.
   It could probably be made a fixed size string of some reasonable length (~128 bytes) without loss
   of generality.

This has a large and extensive ripple of changes that achieve the
goals of this project. Chasing these down and fixing them will be the bulk of the effort. There are
a few others issues to be careful about

*  Some mark is needed to track if there is a body at all - there is a semantic difference between a
   body of length zero and no body at all.

Further Work
============

A new "Format" class could be done which processes the current log style format strings in a generic
and modular way. The body factory would be updated to use these new classes.

Documentation
==============

The following sections are intended as documentation to be added to the |TS| documentation set.

TSHttpTxnApplyLogFormat
=======================

Synopsis
--------

`#include <ts/ts.h>`

.. function:: TSReturnCode TSHttpTxnApplyLogFormat(TSHttpTxn txnp, char const* format, TSIOBuffer out)

Description
-----------

Apply the log :arg:`format` string to the transaction :arg:`txnp` placing the result in the buffer
:arg:`out`. The format is identical to :ref:`custom log formats <custom-logging-fields>`. The result
is appended to :arg:`out` in ASCII format. :arg:`out` must be created by the plugin.

See also
--------

:manpage:`TSIOBufferCreate(3ts)`, :manpage:`TSIOBufferReaderAlloc(3ts)`

TSHttpTxnErrorBodySet
=====================

Sets an error type body to a transaction.

Synopsis
--------

`#include <ts/ts.h>`

.. function:: void TSHttpTxnErrorBodySet(TSHttpTxn txnp, char * buf, size_t buflength, char * mimetype)

Description
-----------

Note that both string arguments must be allocated with :c:func:`TSmalloc` or
:c:func:`TSstrdup`.  The :arg:`mimetype` is optional, and if not provided it
defaults to :literal:`text/html`. Sending an empty string would prevent setting
a content type header (but that is not adviced).

TSIOBufferReader
==================

Traffic Server IO buffer reader API.

Synopsis
--------

`#include <ts/ts.h>`

.. function:: TSIOBufferReader TSIOBufferReaderAlloc(TSIOBuffer bufp)
.. function:: TSIOBufferReader TSIOBufferReaderClone(TSIOBufferReader readerp)
.. function:: void TSIOBufferReaderFree(TSIOBufferReader readerp)
.. function:: void TSIOBufferReaderConsume(TSIOBufferReader readerp, int64_t nbytes)
.. function:: TSIOBufferBlock TSIOBufferReaderStart(TSIOBufferReader readerp)
.. function:: int64_t TSIOBufferReaderAvail(TSIOBufferReader readerp)
.. function:: int TSIOBufferReaderIsAvailAtLeast(TSIOBufferReader, int64_t nbytes)
.. function:: int64_t TSIOBufferReaderRead(TSIOBufferReader reader, const void * buf, int64_t length)
.. function:: int TSIOBufferReaderIterate(TSIOBufferReader reader, TSIOBufferBlockFunc* func, void* context)

.. type:: TSIOBufferBlockFunc

   ``int (*TSIOBufferBlockFunc)(void const* data, int64_t nbytes, void* context)``

   :arg:`data` is the data in the :type:`TSIOBufferBlock` and is :arg:`nbytes` long. :arg:`context` is
   opaque data provided to the API call along with this function and passed on to the function. This
   function should return ``true`` to continue iteration or ``false`` to terminate iteration.

Description
-----------

:type:`TSIOBufferReader` is an read accessor for :type:`TSIOBuffer`. It represents a location in
the contents of the buffer. A buffer can have multiple readers and each reader consumes data in the
buffer independently. Data which for which there are no readers is discarded from the buffer. This
has two very important consequences --

* Data for which there are no readers and no writer will be discarded. In effect this means without
   any readers only the current write buffer block will be maintained, older buffer blocks will
   disappear.
*  Conversely keeping a reader around unused will pin the buffer data in memory. This can be useful or harmful.

A buffer has a fixed amount of possible readers (currently 5) which is determined at compile
time. Reader allocation is fast and cheap until this maxium is reached at which point it fails.

:func:`TSIOBufferReaderAlloc` allocates a reader for the IO buffer :arg:`bufp`. This should only be
      called on a newly allocated buffer. If not the location of the reader in the buffer will be
      indeterminate. Use :func:`TSIOBufferReaderClone` to get another reader if the buffer is
      already in use.

:func:`TSIOBufferReaderClone` allocates a reader and sets its position in the buffer to be the same as :arg:`reader`.

:func:`TSIOBufferReaderFree` de-allocates the reader. Any data referenced only by this reader is
      discarded from the buffer.

:func:`TSIOBufferReaderConsume` advances the position of :arg:`reader` in its IO buffer by the
      the smaller of :arg:`nbytes` and the maximum available in the buffer.

:func:`TSIOBufferReaderStart` returns the IO buffer block containing the position of
:arg:`reader`.

   .. note:: The byte at the position of :arg:`reader` is in the block but is not necessarily the first byte of the block.

:func:`TSIOBufferReaderAvail` returns the number of bytes which :arg:`reader` could consume. That is
      the number of bytes in the IO buffer starting at the current position of :arg:`reader`.

:func:`TSIOBufferReaderIsAvailAtLeast` return ``true`` if the available number of bytes for
      :arg:`reader` is at least :arg:`nbytes`, ``false`` if not. This can be more efficient than
      :func:`TSIOBufferReaderAvail` because the latter must walk all the IO buffer blocks in the IO
      buffer. This function returns as soon as the return value can be determined. In particular a
      value of ``1`` for :arg:`nbytes` means only the first buffer block will be checked.

:func:`TSIOBufferReaderRead` reads data from :arg:`reader`. This first copies data from the IO
      buffer for :arg:`reader` to the target buffer :arg:`bufp`, starting at :arg:`reader`s
      position, and then advances (as with :func:`TSIOBufferReaderConsume`) :arg:`reader`s
      position past the copied data. The amount of data read in this fashion is the smaller of the
      amount of data available in the IO buffer for :arg:`reader` and the size of the target buffer
      (:arg:`length`).

:func:`TSIOBufferReaderIterate` iterates over the blocks for :arg:`reader`. For each block
:arg:`func` is called with with the data for the block and :arg:`context`. The :arg:`context` is an
opaque type to this function and is passed unchanged to :arg:`func`. It is intended to be used as
context for :arg:`func`. If :arg:`func` returns ``false`` the iteration terminates. If :arg:`func`
returns true the block is consumed. The return value for :func:`TSIOBufferReaderIterate` is the
return value from the last call to :arg:`func`.

   .. note:: If it would be a problem for the iteration to consume the data (especially in cases where
             ``false`` might be returned) the reader can be cloned via :func:`TSIOBufferReaderClone` to
             keep the data in the IO buffer and available. If not needed the reader can be destroyed or
             if needed the original reader can be destroyed and replaced by the clone.

.. note:: Destroying a :type:`TSIOBuffer` will de-allocate and destroy all readers for that buffer.



See also
--------

:manpage:`TSIOBufferCreate(3ts)`
