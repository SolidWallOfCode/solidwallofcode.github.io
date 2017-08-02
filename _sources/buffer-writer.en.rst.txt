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

   w.l(" [");
   if (s->txn_conf->insert_request_via_string > 2) { // Highest verbosity
      w.cstr(incoming_via);
   } else {
      w(incoming_via + VIA_CLIENT, VIA_SERVER - VIA_CLIENT);
   }
   w(']');

   // reserve 4 for " []" and 3 for "])".
   if (w.auxSize() > 4 && s->txn_conf->insert_request_via_string > 3) { // Ultra highest verbosity
      w.l(" [");
      w.auxWrite(write_via_protocol_stack(
         w.auxBuffer(), w.auxSize() - 3), true, proto_buf.data(), n_proto));
      w.(']');
   }

   if (w.error()) Warning("VIA: static buffer overflow");

In addition to being shorter and cleaner, the latter is also far more robust in that if the buffer
overflows there will be no memory corruption because :class:`BufferWriter` will avoid doing the
overflowing writes, and stop output at the last element that fits in the target buffer.

If :code:`write_via_protocol_stack` were changed a bit to have the signature ::

   size_t write_via_protocol_stack(BufferWriter& w, bool, ts::StringView * proto int n_proto)

then :code:`BufferWriter::write` can be used to further simplify the code for the VIA output to ::

      w.l(" [")
        .reserve(1).write(&write_via_protocol_stack, true, proto_buf.data(), n_proto)
        .release().c(']');

Issues
++++++

A remaining issue is how non string data is provided to :class:`BufferWriter`. This should be
sufficiently generic that new types can be supported without changing :class:`BufferWriter`. It has
been suggested the standard stream operators be overloaded to work with a :class:`BufferWriter`,
e.g. ::

   BufferWriter& operator << (BufferWriter& w, SomeSpecificType const& t) { ... }

This can be done independently as :class:`BufferWriter` provides sufficient functionality to
implement these stream operators if desired.

Reference
+++++++++

.. class:: BufferWriter

   :class:`BufferWriter` is the abstract base class which defines the basic client interface. This
   is intended to be the reference type used when passing concrete instances rather than having to
   support the distinct types.

   .. function:: template <size_t N> BufferWriter & l(const char (&literal)[N])

      Write :arg:`literal` to the output treating it as a literal string. This will clip the terminating null.

   .. function:: BufferWriter& cstr(const char * str)

      Write :arg:`str` to the output treating it as a C-string (null terminated). This will call
      :code:`strlen` on :arg:`str` to determine the length.

   .. function:: BufferWriter& write(std::function<size_t (BufferWriter&, ...)> f, ...)

      Create a nested :class:`BufferWriter` that operates on the empty part of the current buffer.
      The function :arg:`f` is called and passed the nested :class:`BufferWriter` and the argument
      pack passed to this method.

   .. function:: BufferWriter& reserve(size_t n)

      Restrict the buffer to be :arg:`n` characters less than it is currently. This effectively reserves :arg:`n` characters at the end of the buffer.

   .. function:: BufferWriter & release()

      Release all reserved space.

   .. function :: BufferWriter & release(size_t n)

      Release :arg:`n` characters of space. The released space is always adjacent to the available buffer.

.. class:: FixedBufferWriter : public BufferWriter

   This is a class that implements :class:`BufferWriter` on a fixed buffer.

   .. function:: FixedBufferWriter(void* buffer, size_t length)

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

Futures
+++++++

A planned future extension is a variant of :class:`BufferWriter` that operates on a :code:`MIOBuffer`. This would be very useful in many places that work with :code:`MIOBuffer` instances, most specifically in the body factory logic.
