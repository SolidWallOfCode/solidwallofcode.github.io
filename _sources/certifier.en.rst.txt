.. include:: common.defs

.. highlight:: cpp
.. default-domain:: cpp
.. |Name| replace:: Certifier

|Name|
**********

This plugin performs two basic tasks -

*  Load SSL certificates from file storage on demand. The total number of loaded certificates can be
   configured.
*  Generates SSL certificates on demand.

Generated certificates can be written to file storage for later retrieval.

Description
===========

|Name| manages SSL certificates for |TS|. There are two sources for certificates, certificate files
in the file system and dynamically generated certificates. Dynamic certificates can be stored to
the file system to avoid generating them again.

Configuration
=============

|Name| is a global plugin and is configured by arguments in :file:`plugin.config`.

:literal:`|Name|`

.. program:: |Name|

.. option:: --sign-using <path>

   Specify the signing certificate for dynamically generated certificates. :arg:`path` should be the
   path and file name of the certificate. If it is relative it is relative to the |TS| configuration
   directory. If this is not specified then dynamic certificate generation is disabled.

.. option:: --max <N>

   The maximum number of certificates to keep in memory. If more certificates are loaded the least
   recently used certificates are deleted from memory. This is intended to control the amount of
   memory used by the in memory certificate store.

.. option:: --store <path>

   The directory to use as the root of file system certificate store.

Implementation
==============



Future Work
===========

It might be useful at some point to enable |Name| to fetch certificates remotely as needed.

|Name| should monitor plugin messages to drop a specific certificate from memory. This makes real
time updating simple - the certificate in the file system store is replaced then |Name| signaled to
drop that certificate from the in memory store. The (new) certificate will then be loaded on next use.

At some point there will be a need to have a sweeper that scans the file system store and removes
certificates that have expired before some specified time point. In practice this is likely to
be specified as a duration before the present time.

Notes
=====

The current work on this is in conjunction with a transaction monitor which records data from
transactions. This is used for generating replay data and to monitor third party appliances in
sensitive locations. That funcationality and all of this is wrapped in a single plugin. The best
plan going forward maybe to have the transaction monitor presume the transactions are decrypted and
use |Name| if it is necessary to do dynamic certificate generation.
