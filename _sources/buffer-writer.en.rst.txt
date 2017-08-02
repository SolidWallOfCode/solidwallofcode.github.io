.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _buffer writer:

Buffer Writer
*************

Buffer writer is a project to improve writting text data to buffers. Currently this is done by code that looks like ::

   *via_string++ = ' ';
   *via_string++ = '[';

   // incoming_via can be max MAX_VIA_INDICES+1 long (i.e. around 25 or so)
   if (s->txn_conf->insert_request_via_string > 2) { // Highest verbosity
      via_string += nstrcpy(via_string, incoming_via);
   } else {
      memcpy(via_string, incoming_via + VIA_CLIENT, VIA_SERVER - VIA_CLIENT);
      via_string += VIA_SERVER - VIA_CLIENT;
   }
   *via_string++ = ']';

   // reserve 4 for " []" and 3 for "])".
   if (via_limit - via_string > 4 && s->txn_conf->insert_request_via_string > 3) { // Ultra highest verbosity
      *via_string++ = ' ';
      *via_string++ = '[';
      via_string += write_via_protocol_stack(via_string, via_limit - via_string - 3,
      true, proto_buf.data(), n_proto);
      *via_string++ = ']';
   }

This is ugly, error prone, and unsafe. Most instances of this style just hope there is no buffer overflow. We need to do better.

The idea is to use C++ eleventy features to build a more robust approach. This is based on a family
of classes that control the output buffer and manage the offset tracking. The base is :class:`BufferWriter` which enables writing to the buffer. Assuming there is an instance of a concrete subclass of :class:`BufferWriter` named :code:`w` which controls the :arg:`via_string` buffer, the equivalent of the above code would be ::

   w.write(" [");
   if (s->txn_conf->insert_request_via_string > 2) { // Highest verbosity
      w.write(incoming_via, strlen(incoming_via));
   } else {
      w.write((incoming_via + VIA_CLIENT, VIA_SERVER - VIA_CLIENT);
   }
   w.write(']');

   // reserve 4 for " []" and 3 for "])".
   if (w.remaining() > 4 && s->txn_conf->insert_request_via_string > 3) { // Ultra highest verbosity
      w.write(" [");
      w.clip(1); // reserve a byte for closing bracket.
      w.write(write_via_protocol_stack(
         w.auxBuffer(), w.remaining() - 3), true, proto_buf.data(), n_proto));
      w.extend(1).write(']');
   }

   if (w.error()) Warning("VIA: static buffer overflow");

In addition to being shorter and cleaner, the latter is also far more robust in that if the buffer
overflows there will be no memory corruption because :class:`BufferWriter` will avoid doing the
overflowing writes, and stop output at the last element that fits in the target buffer.

If :code:`write_via_protocol_stack` were changed a bit to have the signature ::

   size_t write_via_protocol_stack(BufferWriter& w, bool, ts::StringView * proto int n_proto)

then the code can be simplified to ::

      w.write(" [");
      w.clip(1); // reserver a byte for closing bracket
      write_via_protocol_stack(w, true, proto_buff.data(), n_proto);
      w.extend(1).write(']'); // always succeeds because we checked for space earlier.

Issues
++++++

A remaining issue is how non string data is provided to :class:`BufferWriter`. This should be
sufficiently generic that new types can be supported without changing :class:`BufferWriter`. It has
been suggested the standard stream operators be overloaded to work with a :class:`BufferWriter`,
e.g. ::

   BufferWriter& operator << (BufferWriter& w, SomeSpecificType const& t) { ... }

This can be done independently as :class:`BufferWriter` provides sufficient functionality to
implement these stream operators if desired. :class:`BufferWriter` itself should have few methods to
actually write to the buffer, just enough to make it possible to write stream operators for it.
Those stream operators should be defined outside of the :class:`BufferWriter` header so that
:class:`BufferWriter` does not need to be updated to add additional IO stream operators for new
types.

For instance if I wanted to make :code:`ts::StringView` work with :class:`BufferWriter` I would add
to the :code:`ts::StringView` header the code ::

   BufferWriter & operator << (BufferWriter & w, ts::StringView & sv) {
      w.write(sv.ptr(), sv.size());
      return w;
   }

Reference
+++++++++

.. class:: BufferWriter

   :class:`BufferWriter` is the abstract base class which defines the basic client interface. This
   is intended to be the reference type used when passing concrete instances rather than having to
   support the distinct types.

   .. function:: BufferWriter & write(void * data, size_t length)

      Write to the buffer starting at :arg:`data` for at most :arg:`length` bytes. If there is not
      enough room to fit all the data, none is written.

   .. function:: BufferWriter & write(string_view str)

      Write the string :arg:`str` to the buffer. If there is not enough room to write the string no
      data is written.

   .. function:: template <size_t N> BufferWriter & write(const char (&literal)[N])

      Write :arg:`literal` to the output treating it as a literal string. This means the size is
      computed by the compiler and the null terminator is discarded. If there is not enough space in
      the buffer no data is written.

   .. function:: BufferWriter & write(char c)

      Write the character :arg:`c` to the buffer. If there is no space in the buffer the character
      is not written.

   .. function:: size_t size() const

      Return the number of valid (written) bytes in the buffer.

   .. function:: size_t remaining() const

      Return the number of available remaining bytes that could be written to the buffer.

   .. function:: BufferWriter & resize(size_t n)

      Set the number of bytes of valid (written) data.

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
