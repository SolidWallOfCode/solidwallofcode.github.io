Traffic Server Layer 7 Working Group
************************************

.. |upstreamer| replace:: strategizer
.. |SM| replace:: :code:`HttpSM`

My view on the Layer 7 Routing working group meetings in Denver, 25-27 Jul 2018.

Configuration
=============

The YAML schema will be updated to be more generic. The "primary" and "secondary" rings will be
removed and replaced with generic "rings" which will be defined in a specific section. The
"strategy" section will be used to create policy that uses the rings. In essence the "ring" section
will describe the elements of the CDN and "strategy" will describe the CDN policy for a specific
node. To make this easier to deploy, some version of John's include support for YAML (which does not have
this natively).

|SM| Interaction
==================

The various strategies in the L7R configuration files will provide a set of "strategies", each
marked with a unique tag. There is a globally configured default strategy tag. During remap, this
can be overridden with a different strategy tag.

When the |SM| decides it needs to send the request to an upstream target, it instantiates a
"|upstreamer|" [#]_ which is a run time stateful object based on a specific strategy. That strategy is
selected by the strategy tag.

The |upstreamer| is responsible for providing transaction ready sessions to upstream targets. The
selection of the target and any handling of layer 3 or 4 connection errors is the resposibility of
the |upstreamer|. When the |SM| receives an open session event (:code:`SESSIOn_READY`) it will send the HTTP request to the
upstream. The |SM| will handle the result or response header for transaction, then report it to the
|upstreamer|. The |upstreamer| then decides on the appropriate next action and informs the |SM|
syncrhonously what that action is. The |SM| performs its required tasks and when it has finished,
reports that to the |upstreamer|. Note that all decisions about how to proceed from the upstream
response is done by the |upstreamer|. This includes not only HTTP responses but also network errors.
Even though the session is open at the time the |SM| receives it, that does not guarantee the
absence of further network problems.

.. uml::
   :align: center
   :caption: Generic Transaction

   entity L7Policy
   actor Strategizer
   actor HttpSM
   participant Upstream

   hide footbox

   HttpSM -\ L7Policy : create "tag"
   L7Policy -/ HttpSM : return: Strategizer
   activate Strategizer
   Strategizer -> HttpSM : SESSION_READY
   note right : Contains session
   HttpSM -> Upstream : HTTP Request
   Upstream -> HttpSM : Upstream Response
   HttpSM -> HttpSM : Read response.
   HttpSM -\ Strategizer : result
   note right : Contains response
   Strategizer -/ HttpSM : return: ACTION
   HttpSM -> HttpSM : process response
   HttpSM -> Strategizer : finish

The interaction is split out in this manner to accomodate the variety of synchronous and
asynchronous operations. In particular if there is an HTTP reponse from the upstream then the |SM|
needs to be able to handle it in an appropriate way. However, that depends on what the next action
as decided by the |upstreamer|. Because the |upstreamer| next action can depend on potentially long
lived asynchronous operations, waiting for such operations to complete is therefore not feasible.
Instead the |upstreamer| must respond synchronously to the response with an action code that
describes what the |upstreamer| will do when the |SM| has finished its operations. It is assumed
that while the next action may take a long time to perform, *determining* the next action should be
a fast computation.

The current valid actions are

``DONE``
   The result is a final result that should be sent on to the user agent.

``RETRY``
   The transaction was a failure of some sort but not a permanent one. The |upstreamer| will prepare another
   session after the |SM| has finished cleaning up the current transaction.

This is clearer with specific examples.

First consider the case when the upstream request is successful. It is unacceptable to wait for
strategy completion to send data to the user agent - that flow needs to start as soon as possible.
The |SM| can no longer know if that is correct without consulting the |upstreamer| therefore that
must be a synchronous call.

.. uml::
   :align: center
   :caption: Successful Transaction

   entity L7Policy
   actor Strategizer
   actor HttpSM
   participant Upstream

   hide footbox

   HttpSM -\ L7Policy : create "tag"
   L7Policy -/ HttpSM : return: Strategizer
   activate Strategizer
   Strategizer -> HttpSM : SESSION_READY
   note right : Contains session
   HttpSM -> Upstream : HTTP Request
   Upstream -> HttpSM : ""200 OK""
   HttpSM -> HttpSM : Read response header.
   HttpSM -\ Strategizer : result
   note right : HTTP 200 Response
   Strategizer -/ HttpSM : return: DONE
   HttpSM -> HttpSM : Read upstream response,\nsend to user agent.
   ...
   HttpSM -> Strategizer : finish

Even in an HTTP failure there is still work for the |SM|. In this case, for a ``404`` response
status the |upstreamer| will fail over to a different upstream. This behavior is used to probe
multiple upstream pods for specific content, a ``404`` indicating the next target should be tried
rather than returning the response to the user agent. The |SM| must drain any body from the response
but needs to know, while draining, that the body will not be returned to the user agent.

.. uml::
   :align: center
   :caption: Fail / Retry Transaction

   entity L7Policy
   actor Strategizer
   actor HttpSM
   participant Upstream

   hide footbox

   HttpSM -\ L7Policy : create "tag"
   L7Policy -/ HttpSM : return: Strategizer
   activate Strategizer
   Strategizer -> HttpSM : SESSION_READY
   note right : Contains session
   HttpSM -> Upstream : HTTP Request
   Upstream -> HttpSM : ""404 NOT FOUND""
   HttpSM -> HttpSM : Read response header.
   HttpSM -\ Strategizer : result
   note right : HTTP 404 Response
   Strategizer -/ HttpSM : return: RETRY
   HttpSM -> HttpSM : Drain upstream response body
   ...
   HttpSM -> Strategizer : finish
   ...
   Strategizer -> HttpSM : SESSION_READY
   == As 200 OK case ==

Another view of the activity demonstrates the "co-routine" like nature of the interaction. Both the |SM| and |upstreamer| perform blocking operations where the other side must wait for an event or signal to indication the operation has completed. The basic logic is

.. uml::
   :align: center
   :caption: Upstream session logic


   partition HttpSM {
      (*) --> "Determine strategy tag"
   }

   partition Strategy {
      "Determine strategy tag" --> "Create strategizer instance"
      "Create strategizer instance" -u-> "Connect upstream"
      "Connect upstream" --> "Event: SESSION_READY"
   }

   partition HttpSM {
      "Event: SESSION_READY" -l->  if "send request" then
         -->[net error] ===RESULT===
      else
         -->[success] "Read Response Header"
         -> ===RESULT===
      endif
   }

   partition Strategy {
      --> "Compute next action"
   }

   partition HttpSM {
      if "ACTION" then
         -->[DONE] "Forward to User Agent"
      else
         -->[RETRY] "Drain session"
      endif
   }

   partition Strategy {
      "Drain session" -->[finish()] "Connect upstream"
      "Forward to User Agent" -down->[finish()] "Destruct"
   }

   partition HttpSM {
      "Destruct" --> (*)
   }


The state sequence in the |SM| is much simplified by this work. In particular, the
nemesis of redirection or other upstream connection failures requiring rolling back the |SM| state
will be avoided. Instead there is a simpler loop which spans a much smaller set of states.

.. uml::
   :align: center
   :caption: State Machine Upstream States

   [*] -> ServerOpen
   ServerOpen -> WaitForTransaction
   WaitForTransaction --> HandleServerTransaction : SESSION_READY
   HandleServerTransaction --> ReadResponseHeader
   ReadResponseHeader --> HandleResponseHeader : HTTP response
   ReadResponseHeader -> ReportServerResponse : network error
   ReportServerResponse -> DrainResponse : ACTION: RETRY
   DrainResponse --> WaitForTransaction
   HandleResponseHeader --> ReportServerResponse
   ReportServerResponse --> HandleServerResponse : ACTION: DONE

   state ServerOpen : Create Strategizer
   HandleServerTransaction : Send request
   ReadResponseHeader : Wait for upstream response
   HandleResponseHeader : Read and validate header
   ReportServerResponse: Report to stategizer\nGet next action
   HandleServerResponse : Send response to User Agent\nSignal strategizer
   DrainResponse : Drain reponse\nSignal strategizer

Next Steps
==========

In my view the key next steps are first, to get :code:`Extendible` committed to master. After that
it was agreed the next deliverable would be a plugin or plugins that would emulate the current
manual host status marking using :code:`Extendible`. This would enable

*  Valdating the :code:`Extendible` API.

*  Experiment with external tool to :code:`Extendible` data mechanisms. That is, how can external
   data be pused in to host and IP address records?

*  Validate that :code:`Extendible` data can be used to manipulate upstream selection.

I think this will also be the keystone of moving forward with :code:`Extendible` as doing things
similar to existing code is much easier than building thing ex nihilo. It might even be reasonable,
as a temporary expedient, to have the core use :code:`Extendible` to create the data for these
purposes and leave the plugin (which will reuquire much more in the way of infrastructure changes)
until later. There is a reasonable chance that the core will end up using :code:`Extendible`
internally, because it makes the overall code base much more modular.

Open Issues
===========

There are still a few open issues which aren't fully resolved.

Setting upstream address
   It is a requirement, for particularly for transparent networking, to be able to explicitly set
   the IP addres of the upstream, both in the core and using the plugin API call
   :code:`TSHttpTxnSetTargetAddr`.

Self Marking
   There must be a descriptor that says "the remapped upstream" so that no explicit upstream hosts
   need to be defined. This is needed for the default routing situation and for forward proxying.

Visions
=======

:code:`Extendible` was quite a hit and there was a push to use it in other situations, in particular
with the |SM|. I pushed back on that because I think updating the |SM| would be a major change and
therefore we should bake :code:`Extendible` a bit more to make sure the API is clean and the code
reliable.

With regard to the use of :code:`Extendible` in |SM| (something Leif and Vijay were giddy about) I
have always thought this was in the long run a good idea, but changing the |SM| is always fraught. I
prefer to get some experience with :code:`Extendible` before making such a major change. Past that
the purpose of this would be to replace the current transaction arguments with :code:`Extendible`.
An issue with this is remap plugins. These have two properties that make it more difficult to use
:code:`Extendible`.

*  Different plugins per remap rule using different sets of extended data.

*  Reloading, which is another ongoing project.

For various reasons (including the nature of the proxy allocators) it is not feasible to have
instances of a class with different extendible schemas. As a result it will probably be necessary to
store a schema per remap rule and dynamically allocate the storage block. This could be justified in
that it is better to allocate once per |SM| than once per remap plugin that needs per transaction
storage. If this isn't sufficient it might be needful to use an :code:`IOBufferBlock`. Given all the
other allocations that go on in the |SM| I suspect reducing that will more than compensate for using
:code:`Extendible` in the |SM|, especially if combined with :code:`MemArena` for general allocation.

Such a change to |SM| will almost require similar changes to :code:`HttpClientSession` and
:code:`AnnotatedConnection`. The reasons for this are the same as for the |SM|, in that each
supports an array of :code:`< void * >` which are used by plugins. In these cases the use is for
global plugins and therefore can be done statically rather than dynamically.

.. rubric:: Footnotes

.. [#] This is a stupid name but no one else would provide a better one, which frankly shouldn't be challenging.
