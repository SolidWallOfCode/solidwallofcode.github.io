.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp

.. _testing:

Testing
*******

I have a number of additional projects to improve the AuTest framework for use with |TS|.

Pure Replay testing
===================

It should be possible to have run tests using only a replay file and potentially some configuration
information, without any Python code at all. This requires the micro-DNS server.

Header Comparators
==================

Frequently the literalness of the gold testing is a problem. AUTest should have comparators that are specialized for HTTP headers. This could be worked in to the replay file format with metadata indicating which headers should be literal, vs. present/absent vs. irrelevant. Replay file based testing would greatly benefit from this.

Traffic Capture / Sampling
==========================

At one point the technology existed to capture traffic for traffic replay. This should be
resurrected and improved. After discussion among the group it seems that for replay purposes literal
content is unnecessary, only the size matters. It is still debated if full capture is useful for
debugging purposes. My opinion is it is not and complicates the capture process unnecessarily.

I think in terms of building out the replay tooling we need a plugin that can to some extent record
traffic. The data recorded should be in the replay file format and contain

*  Each session and the transactions in the session.
*  Timestamps.
*  The four headers.
*  The protocol stack for the user agent.
*  The transaction count for the outbound session.
*  The content block sizes.

The plugin should record samples rather than all sessions as the latter isn't possible on most
production machines. The plugin should have some sampling percentage and sample that fraction of
*sessions*. For content, the actual content isn't particularly useful but the I/O operation sizes
could be significant. The plugin should record the number of bytes in each chunk of data that passes
through the transaction in both directions.

Production Verification
=======================

It would be nice to be able to run AUTest against live production systems to verify behavior,
particularly with regard to paranoid requirements.

Configuration Verification
==========================

AuTest could be integrated with configuration management. This would involve setting up a |TS|
instance with a proposed configuration deployment in a non-production environment and then verifying
various functional properties. This would similar to production verification but presumably more
thorough and focused on operatoinsl needs.

Replay File Format
==================

The replay file is basically a list of sessions. Each session contains connection information and a
list of transactions. Each transaction contains some meta data and set of our transaction headers.
These are the inbound request, the outbound request, the upstream response, and the inbound
response.

Each transaction can also have a UUID which can be used to specify the appropriate response. The
UUID can be sent along with the transaction as an extra header if needed. Otherwise the transactions
must be matched up based on other data in the request.

*  `File schema <_static/json/replay-file.json>`__.
*  `Example <_static/json/replay-example.json>`__.

Diagrams
========

.. graphviz::

   digraph {
       testing_project [label="Testing" shape=folder];
       replay_file_design [label="Replay File\nDesign" shape=rect style=rounded];
       replay_file_testing [label="Replay File\nBased Testing" shape=rect style=rounded]
       configuration_verification [label="Configuration\nVerification" shape=rect style=rounded]
       production_verification [label="Production\nVerification" shape=rect style=rounded]

       testing_project -> {replay_file_design};
       replay_file_design -> {production_verification, configuration_verification, replay_file_testing}
   }
