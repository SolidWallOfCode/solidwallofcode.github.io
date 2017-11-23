.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _reference:

Diagrams
*************************

Process start up

.. uml::

   @startuml

   :netProcessor.init();
   :init_HttpProxyServer();
   :eventProcessor::start();
   note right: ET_NET threads start here
   :plugin_init();
   note right: Global plugin initialization
   :cacheProcessor.start();
   fork
   :udpNet::start();
   :init_accept_HttpProxyServer();
   :TS_EVENT_LIFECYCLE_PORTS_INITIALIZED>
   if (cache_wait) then (ready)
   :start_HttpProxyServer();
   :TS_EVENT_LIFECYCLE_PORTS_READY>
   else (not ready)
   :cache_wait = waited;
   endif
   :tasksProcessor.start();
   fork again
   :Cache Initialization;
   if (cache_waited) then (waited)
   :start_HttpProxyServer();
   :TS_EVENT_LIFECYCLE_PORTS_READY>
   else (not ready)
   :cache_waited = ready;
   endif
   :TS_EVENT_LIFECYCLE_CACHE_READY>
   end fork
   @enduml
