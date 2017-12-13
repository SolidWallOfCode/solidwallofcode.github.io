.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _BufferWriter:

BufferWriter
*************

:class:`BufferWriter` is a class to write text to a buffer. The design goals are to be fast and
provide buffer overrrun protection. There is a large amount of code in |TS| that currently uses
:code:`memcpy` and similar mechanisms for fast string generation. :class:`BufferWriter` should be
as almost as fast while simplifying code and preventing memory corruption.

For example, error prone code that looks like

.. code-block:: cpp

   char new_via_string[1024]; // 512-bytes for hostname+via string, 512-bytes for the debug info
   char * via_string = new_via_string;
   char * via_limit  = via_string + sizeof(new_via_string);

   // ...

   * via_string++ = ' ';
   * via_string++ = '[';

   // incoming_via can be max MAX_VIA_INDICES+1 long (i.e. around 25 or so)
   if (s->txn_conf->insert_request_via_string > 2) { // Highest verbosity
      via_string += nstrcpy(via_string, incoming_via);
   } else {
      memcpy(via_string, incoming_via + VIA_CLIENT, VIA_SERVER - VIA_CLIENT);
      via_string += VIA_SERVER - VIA_CLIENT;
   }
   *via_string++ = ']';

becomes

.. code-block:: cpp

   ts::LocalBufferWriter<1024> w; // 1K internal buffer.

   // ...

   w << " [";
   if (s->txn_conf->insert_request_via_string > 2) { // Highest verbosity
      w << incoming_via;
   } else {
      w << ts::string_view{incoming_via + VIA_CLIENT, VIA_SERVER - VIA_CLIENT};
   }
   w << ']';

Note that in addition there will be no overrun on the memory buffer in :arg:`w`, in strong contrast
to the original code.

Description
+++++++++++

:class:`BufferWriter` is an abstract super class, in the style of :code:`std::ostream`. There are several subclasses for various use cases.

:class:`FixedBufferWriter` writes to an external provided buffer of a fixed length. The buffer must
be provided to the constructor. This will generally be used in a function where the target buffer is
external to the function.

:class:`LocalBufferWriter` is a templated class whose template argument is the size of an internal buffer.
This is useful when the buffer is local to a function and the results will be transferred from the buffer to other storage after the output is assembled. E.g. code such as

.. code-block:: cpp

   char buff[1024];
   ts::FixedBufferWriter w(buff, sizeof(buff));

is better done as

.. code-block:: cpp

   ts::LocalBufferWriter<1024> w;

Writing
-------

The basic mechanism for writing to a :class:`BufferWriter` is :func:`BufferWriter::write`.
This is an overloaded method for a character (:code:`char`), a buffer (:code:`void *, size_t`)
and a string view (:code:`ts::string_view`).

On top of this mechanism are stream operators in the style of C++ stream I/O. The basic template
is

.. code-block:: cpp

   template < typename T > ts::BufferWriter& operator << (ts::BufferWriter& w, T const& t);

Most basic types are overloaded and it is easy to extend. For instance, to make :code:`ts::TextView` work with :class:`BufferWriter`, the code would be

.. code-block:: cpp

   ts::BufferWriter & operator << (ts::BufferWriter & w, ts::TextView const & sv) {
      w.write(sv.data(), sv.size());
      return w;
   }

Reading
-------

The data in the buffer can be extracted using :func:`BufferWriter::data`. This and :func:`BufferWriter::size`
return a pointer to the start of the buffer and the amount of data written to the buffer. Calling :func:`BufferWriter::error` will indicate if more data than space available was written. :func:`BufferWriter::extent` returns the amount of data written to the :class:`BufferWriter`. This can be used in a two pass style with a small buffer to determine the buffer size required for the full output.

Advanced
--------

The :func:`BufferWriter::clip` and :func:`BufferWriter::extend` methods can be used to reserve space in the buffer. A common use case for this is to guarantee matching delimiters in output if buffer space is exhausted.
:func:`BufferWriter::clip` can be used to temporarily reduce the buffer size by an amount large enough to hold
the terminal delimiter. After writing the contained output, :func:`BufferWriter::extend` can be used to restore
the capacity and then output the terminal delimiter.

.. warning:: **Never** call :func:`BufferWriter::extend` without previoiusly calling :func:`BufferWriter::clip` and always pass the same argument value.

:func:`BufferWriter::capacity` returns the amount of buffer space not yet consumed.

:func:`BufferWriter::auxBuffer` returns a pointer to the first byte of the buffer not yet used. This is useful to do speculative output. A new :class:`BufferWriter` instance can be constructed with

.. code-block:: cpp

   ts::FixedBufferWriter subw(w.auxBuffer(), w.remaining());

Output can be written to :arg:`subw`. If successful, then :code:`w.write(subw.size())` will add that output to the main buffer. If there is an error then :arg:`subw` can be ignored and some suitable error output written to :arg:`w` instead. A common use case is to verify there is sufficient space in the buffer and create a "not enough space" message if not. E.g.

.. code-block:: cpp

   ts::FixedBufferWriter subw(w.auxBuffer(), w.remaining());
   this->write_some_output(subw);
   if (!subw.error()) w.write(subw.size());
   else w << "Insufficient space"_sv;

Reference
+++++++++

.. class:: BufferWriter

   :class:`BufferWriter` is the abstract base class which defines the basic client interface. This
   is intended to be the reference type used when passing concrete instances rather than having to
   support the distinct types.

   .. function:: BufferWriter & write(void * data, size_t length)

      Write to the buffer starting at :arg:`data` for at most :arg:`length` bytes. If there is not
      enough room to fit all the data, none is written.

   .. function:: BufferWriter & write(ts::string_view str)

      Write the string :arg:`str` to the buffer. If there is not enough room to write the string no
      data is written.

   .. function:: template <size_t N> BufferWriter & write(const char (&literal)[N])

      Write :arg:`literal` to the output treating it as a literal string. This means the size is
      computed by the compiler and the null terminator is discarded. If there is not enough space in
      the buffer no data is written.

   .. function:: BufferWriter & write(char c)

      Write the character :arg:`c` to the buffer. If there is no space in the buffer the character
      is not written.

   .. function:: char * data() const

      Return a pointer to start of the buffer.

   .. function:: size_t size() const

      Return the number of valid (written) bytes in the buffer.

   .. function:: size_t remaining() const

      Return the number of available remaining bytes that could be written to the buffer.

   .. function:: size_t capacity() const

      Return the number of bytes in the buffer.

   .. function:: char * auxBuffer() const

      Return a pointer to the first byte in the buffer not yet consumed.

   .. function:: BufferWriter & clip(size_t n)

      Reduce the available space by :arg:`n` bytes.

   .. function:: BufferWriter & extend(size_t n)

      Increase the available space by :arg:`n` bytes. Extreme care must be used with this method as
      :class:`BufferWriter` will trust the argument, having no way to verify it. In general this
      should only be used after calling :func:`BufferWriter::clip` and passing the same value.
      Together these allow the buffer to be temporarily reduced to reserve space for the trailing
      element of a required pair of output strings, e.g. making sure a closing quote can be written
      even if part of the string is not.

   .. function:: bool error() const

      Return :code:`true` if the buffer has overflowed from writing, :code:`false` if not.

   .. function:: size_t extent() const

      Return the total number of bytes in all attempted writes to this buffer. This value allows a
      successful retry in case of overflow, presuming the output data doesn't change. This works
      well with the standard "try before you buy" approach of attempting to write output, counting
      the characters needed, then allocating a sufficiently sized buffer and actually writing.

.. class:: FixedBufferWriter : public BufferWriter

   This is a class that implements :class:`BufferWriter` on a fixed buffer.

   .. function:: FixedBufferWriter(void * buffer, size_t length)

      Construct an instance that will write to :arg:`buffer` at most :arg:`length` bytes. Individual
      items are either written or not - if the item doesn't fit nothing is written and the instance
      goes in to an error state the prevents additional output. This is so that if a large item
      doesn't fit a following smaller item isn't output out of order.

.. class:: template < size_t N > LocalBufferWriter : public BufferWriter

   This is a convenience class which is a wrapper on :class:`FixedBufferWriter` which creates a
   buffer as a member rather than having an external buffer that is passed to the instance. The
   buffer is :arg:`N` bytes long. The point of this is to replace code like this::

      char buff[1024];
      FixedBufferWriter w(buff, sizeof(buff));

   with::

      LocalBufferWriter<1024> w;

   Having the buffer size in only one place promotes more robust code.

Futures
+++++++

A planned future extension is a variant of :class:`BufferWriter` that operates on a
:code:`MIOBuffer`. This would be very useful in many places that work with :code:`MIOBuffer`
instances, most specifically in the body factory logic.
