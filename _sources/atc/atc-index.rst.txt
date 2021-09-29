.. include:: ../common.defs

**********************
Apache Traffic Control
**********************

My notes on creating a prototype CDN using ATC and ATS. All of this is specifically for CentOS 8.

Generic Preparation
*******************

Certain bits of data applying to the overall CDN must be decided before installing. This includes

*  A domain name for the CDN.

*  A domain name for the CDN infrastructure (which can be the same as the CDN).

*  Host names for the hosts on which Traffic Control components will installed.

*  Admin user name and password.

*  Vault user name and password.

Various bits of the install will ask for these values, it is critical to be consistent.

It is possible to decide on an install path but unless it is absolutely critical to have the
installs in a specific location it is *much* easier to use the "/opt" default built in to Traffic
Control. Similarly for the "traffic_ops" and "traffic_vault" users in the databases - I have
verified it is possible to change them but it's a lot of work for very little reward. I'd avoid it.

Traffic Ops
***********

The Traffic Ops host will host both the Postgresql database and the ATC "traffic_ops" package.
"traffic_ops" is the central engine of ATC - most other components connect to "traffic_ops" and
treat it as the source of truth. The current recommendation is to install the database and
"traffic_ops" on the same host to avoid having to configure external access to the database.

#. Install Postgresql. An install script can be obtained from `here <https://www.postgresql.org/download/linux/redhat/>`__
   or this one can be used (which was obtained from that webpage).

   .. code-block::

      # Install the repository RPM:
      sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm

      # Disable the built-in PostgreSQL module:
      sudo dnf -qy module disable postgresql

      # Install PostgreSQL:
      sudo dnf install -y postgresql13-server

      # Optionally initialize the database and enable automatic start:
      sudo /usr/pgsql-13/bin/postgresql-13-setup initdb
      sudo systemctl enable postgresql-13
      sudo systemctl start postgresql-13

   Differences:

   *  No configuration to enable access from other hosts. All access will be from this host.
   *  Root access is via user id, not password.

   As of Traffic Control 6.0.0 and Postgresql 13 (at this time, the stable version of each), there is
   a mismatch between how Postgresql and Traffic Control expect to perform password authentication.
   To fix this, Postgresql needs to be adjusted to use an older method.

   Edit "/var/lib/pgsql/13/data/postgresql.conf" and make this change

   .. code-block::

      password_encryption = md5       # md5 or scram-sha-256

   It may or not be necessary to also change "/var/lib/pgsql/13/data/pg_hba.conf" to enable "md5"
   password authentication. This involves changing "scram-sha-256" to "md5".

   .. code-block::

      # IPv4 local connections:
      host    all             all             127.0.0.1/32            md5
      # IPv6 local connections:
      host    all             all             ::1/128                 md5

   If this is not done, you may see this error from "traffic_ops"

   .. code-block::

      pq: unknown authentication response: 10

#. Install `EPEL <https://fedoraproject.org/wiki/EPEL>`__ which provides some of the "traffic_ops" requirements.

   .. code-block::

      sudo dnf install -y epel-release

#. Install the "traffic_ops" RPM. This should automatically install all dependencies.

   .. code-block::

      sudo dnf install --enablerepo=powertools --enablerepo=epel -y traffic_ops-6.0.0-11512.e4469f55.el8.x86_64.rpm

#. Initialize the databases.

   Run a shell as the "postgres" user.

   .. code-block::

      sudo su -c "bash -l" - postgress

   Create the "traffic_ops" user, in this case with the password "vgl". In production this should be
   something a bit more secure. Because of the fragility of the install process, a "traffic_vault" user
   must also be created or the Traffic Vault database will not be populated [#]_.

   .. code-block::

      psql
      CREATE USER traffic_ops with encrypted password 'vgl';
      CREATE USER traffic_vault with encrypted password 'vgl';
      \q

   If doing this by hand the terminal output should look like

   .. code-block::

      $ psql
      psql (13.4)
      Type "help" for help.

      postgres=# CREATE USER traffic_ops with encrypted password 'vgl';
      CREATE ROLE
      postgres=# CREATE USER traffic_vault with encrypted password 'vgl';
      CREATE ROLE
      postgres=# \q
      $

   Then at the shell command prompt (*not* in the ``psql`` environment) create the databases.

   .. code-block::

      createdb traffic_ops --owner traffic_ops
      createdb traffic_vault --owner traffic_vault

   Then ``exit`` from the "postgres" user shell.

#. Finalization

   The script "/opt/traffic_ops/install/bin/postinstall" needs to be invoked as root. Unfortunately
   the check for postgresql superuser may not work. Line 38 may need to be changed to

   .. code-block::

       if [[ ! $(su -c 'psql -c "show is_superuser"' - postgres </dev/null 2>/dev/null) =~ 'on' ]]; then

   At some point I will submit a PR to fix this.

   For the "secrets", this is still a bit of a mystery to me.

#. Network Configuration

   TLS requires a certificates, which consist of a certificate file and corresponding a key. The
   certificate is public and the key file private, the latter being essentially the "password" for
   the certifcate. These can be anywhere as long as they are accessible to the "traffic_ops"
   process. Do make sure the key file has restricted read access. The best locations are either in
   "/etc/pki" or somewhere in the "traffic_ops" install directory (generally "/opt/traffic_ops").
   The location of these files is stored in the "cdn.conf" file in "/opt/traffic_ops/app/conf". For
   legacy reasons these are referenced by editing the URL for the ``listen`` key under the
   ``hypnotoad`` key. In the query string of this URL are two keys, "cert" and "key". The "cert"
   value must be the absolute path to the certificate (".pem") file. The "key" value must the
   absolute path to the key (".key") file.

   Components that connect to Traffic Ops require the hostname to be in the certificate. The hostname
   to check is pulled from the connection URL - that is, if Traffic Portal is set to connect to

   .. code-block::

      https://atc-ops

   then the hostname "atc-ops" must be in the certificate. This can be done by making it the common
   name or by putting it as a ``DNS`` entry in the SAN data. See the certificate generation scripts
   for examples of how to do this.

   To allow external connections from Traffic Portal the TLS port must opened in the firewall. This is
   done with

   .. code-block::

      sudo firewall-cmd --permanent --zone=public --add-service=https
      sudo firewall-cmd --reload

.. rubric:: References

* `Installing Traffic Ops <https://traffic-control-cdn.readthedocs.io/en/latest/admin/traffic_ops.html>`__
* `Installing Postgresql <https://www.postgresql.org/download/linxu/redhat>`__

.. rubric:: Notes

postgresql
   ``createdb`` is a command line tool, not an SQL command used inside the ``psql`` context.

   Switching user id to "postgres" is needed only for root database control. This setup will be
   listening on the standard postgresql port on localhost (IPv4 and IPv6) which is what "traffic_ops"
   will use to perform its operations, logging as the user created for it.

   Do *not* install the standard postgresql package - that causes problems. One effect seems to be
   the command line utilities don't get installed in "/usr/bin" which causes other problems.

Traffic Portal
**************

The base instructions worked.

.. code-block::

   curl --silent --location https://rpm.nodesource.com/setup_14.x | sudo bash -
   sudo dnf install -y nodejs
   sudo dnf install -y /tmp/traffic_portal-6.0.0-11512.e4469f55.el8.x86_64.rpm

It is expected that Traffic Portal will accept incoming browser requests and then make requests
to the Traffic Ops host for data.

The Traffic Portal listening ports must be opened in the CentOS firewall.

.. code-block::

      sudo firewall-cmd --permanent --zone=public --add-service=http
      sudo firewall-cmd --permanent --zone=public --add-service=https
      sudo firewall-cmd --reload

.. rubric:: References

* `Installing Traffic Portal <https://traffic-control-cdn.readthedocs.io/en/latest/admin/traffic_portal/installation.html>`__

Traffic Server
**************

This requires building an RPM for Traffic Server. There is a base ".spec" file in
"tools/packages/trafficserver.spec". This requires some modification to work correctly. The
primary reason is to make the paths compatible with ``t3c``.

To build the RPM several packages are needed on the build machine.

Base RPM tools.

.. code-block::

   sudo dnf install -y epel-release
   sudo dnf install -y rpmdevtools

Packages required by the ATS ".spec" file.

.. code-block::

   sudo dnf --enablerepo=epel --enablerepo=powertools install -y expat-devel hwloc-devel libcurl-devel ncurses-devel

The ``rpmbuild`` command will put the output in "~/rpmbuild/RPMS" by default. This can be changed in
the ".spec" file by adding

   .. code-block::

      %_topdir /path

The underlying directory structure (e.g. that the output goes in to "_topdir/RPMS" doesn't seem
overridable.

The "astats_over_http" plugin is required to enable monitoring and statistics. This is no longer
part of the standard Traffic Server installation (it's been replaced by "stats_over_http" which is
just enough different to not work). This will need to be obtained from the Traffic Control repository.
I need to work out the best way to build and package it - unfortunately building requires an
install of at least the development package from Traffic Server. For now it's a single binary
which I copy by hand.

Administration
**************

Before configuring any Traffic Server instances the CDN must be created in Traffic Ops.

It is not possible to immediately create those "servers" (records for ATS instances). Information
about the CDN must be created first. The steps for that are

#. Define a "division".

#. Define a "region".

#. Define a "location" (aka "physical location").

#. Define a "cache group".

#. Define a "profile" of type ``ATS_PROFILE``. It works best if at least two are created, one for
   mid-tier instances and another for edge.

This sets up the basic CDN. Local configuration of Traffic Server depends on "Parameters". These
must be defined and then attached to the profile used to configure the Traffic Server instance. The
Parameters required to configure a Traffic Server instance are

===================================== ============== ===========================================
Name                                  Config File    Value
===================================== ============== ===========================================
path                                  astats.config  _astats
record_types                          astats.config   122
allow_ip                              astats.config  127.0.0.1
location                              astats.config  /opt/trafficserver/etc/trafficserver
traffic_server                        package        9.1.0-el8.x86_64
CONFIG proxy.config.http.server_ports records.config 80 80:ipv6
===================================== ============== ===========================================

There are several types of Parameters, apparently distinguished by the "Config File" value and
sometimes the name. Generically each Parameter decribes a value / line to be placed in a configuration
file on the local system when running ``t3c``. If other values in "records.config" are needed each
is added as a Parameter with the name of the configuration option, the value, and the config file
"records.config", and then the Parameter attached to the profiles which in turn are attached to
Servers which correspond to Traffic Server instances. ``t3c`` works back from the host name to
the Server to the Profile to the Parameters when generating local configuration files.

There are some special cases.

A Config File of "package" indicates this is a package to install. The actual package name is
generated by concatenating the "name" and "value" of the Parameter. In this case, the package
"traffic_server.9.1.0-el8.x86_64" is to be installed. This calls out to ``yum`` to do the work
therefore if the package is already installed it is skipped.

A Parameter Name of "location" indicate the location of the configuration file, not any of its
contents. This enables moving configuration files around, but is *required* for any configuration
file that is not part of the standard Traffic Server install. The common case for this is
plugins which require a configuration file - that must be specified in a Parameter.

Having created these Parameters, assign them to the Traffic Server profiles created earlier.
Although these Profiles will start out identical they will rapidly diverge as more detailed
configuration is needed.

Configuration
=============

Current ``t3c`` command.

.. code-block::

      sudo t3c apply --verbose --run-mode=badass \
                     --cache-host-name=mid-ts \
                     --traffic-ops-user=admin \
                     --traffic-ops-password=vgl \
                     --traffic-ops-url=https://atc-ops \
                     --git=no \

The ``--cache-host`` is the key used to look up the Server in Traffic Ops which in turn controls the
Profile used to generate the configuration files.

Tools
*****

Communications should be over TLS and that requires certificates. For the prototype I generated my
own root certificate and used it to sign the deployed certificates. It is expected that instead of
answering prompts, the scripts will be edited to put the appropriate values in the certificates.

Note - this is fine for proof of concept and prototyping, but not for actual production use.

Both scripts depend on a file name "openssl.cnf" which is defined by OpenSSL. This is the base
content, which should be tweaked as needed for local conditions.

.. code-block::

   # SPDX-License-Identifier: Apache-2.0

   # Required environment variables
   # PREFIX - used to set the common name.
   # DN - used to select a root or signed distinguished name. Must be "CA" or "Signed".

   [ ca ]
   default_ca = CA_default

   [ CA_default ]
   new_certs_dir = .

   [ req ]
   # Disable prompting so the commands use data from here!
   prompt = no
   distinguished_name = DN-${ENV::DN}

   [DN-Signed]
   CN = base.ex:role.${ENV::PREFIX}
   O = TxnBox
   C = S3
   ST = Amber
   L = Kolvir

   [DN-CA]
   CN = TxnBox CA ${ENV::PREFIX}
   O = TxnBox
   C = S3
   ST = Confusion
   L = Dazed


The script to generate the root certificate.

.. code-block::

   # SPDX-License-Identifier: Apache-2.0
   # Generate root certificate.
   # Note - after running this, no existing signed certificate will work with the new root CA.

   # the openssl.cnf file requires two environment variables
   # PREFIX - the certificate prexifx. This is used to decorate the distinguished name.
   # DN - this must be "CA" for root certs and "Signed" for signed certs.

   # Set serial number for signed certs.
   echo 1 > ca-serial-no.txt

   # Generate the root cert key.
   rm -f vgl-ca.key
   openssl genrsa -passout pass:claudio -des3 -out vgl-ca.key 2048

   # Create the root cert.
   rm -f vgl-ca.pem
   PREFIX=vgl DN=CA openssl req -passin pass:claudio -config ./openssl.cnf -x509 -new -nodes \
      -key vgl-ca.key -sha256 -days 36500 -batch -out vgl-ca.pem

Once the root certificate has been generated, simple signed certificates for deployment are
generated with this script.

.. code-block::

   # SPDX-License-Identifier: Apache-2.0
   # Bash script to generate signed certificates.

   # the openssl.cnf file requires two environment variables
   # PREFIX - the certificate prefix. This is used to decorate the distinguished name.
   # DN - this must be "CA" for root certs and "Signed" for signed certs.

   let N=$(cat ca-serial-no.txt) # signed certificate serial number.

   # Generate key for the signed cert.
   name="vgl-signed-${N}"
   key_file="${name}.key"
   cert_file="${name}.pem"

   rm -f ${key_file}
   PREFIX=vgl DN=Signed openssl genrsa -passout pass:claudio -out ${key_file} 2048

   # Generate the CSR for the signed cert.
   rm -f tmp.csr
   PREFIX=vgl DN=Signed openssl req -passin pass:claudio -config ./openssl.cnf -new -key ${key_file} -batch -out tmp.csr

   # Generate the signed certificate.
   rm -f ${cert_file}
   PREFIX=vgl DN=Signed openssl x509 -passin pass:claudio -req -in tmp.csr -CA vgl-ca.pem -CAkey vgl-ca.key \
        -set_serial 0$N -out ${cert_file} -days 36500 -sha256

   # cleanup.
   rm -f tmp.csr
   let N=N+1
   echo ${N} > ca-serial-no.txt

If the common name is based on the CDN and not the Traffic Ops host, an alternate script can be
used which takes as a single argument the host name to embed in the SAN data. This scripts uses
the domain "solidwallofcode.com" which should be changed to the domain being used for the CDN
infrastructure.

.. code-block::

   # SPDX-License-Identifier: Apache-2.0
   # Bash script to generate signed certificate for a specific host.

   host="$1"
   if [ -z "$host" ] ; then
      echo "Usage: $(basename $0) host"
      exit 1
   fi

   # the openssl.cnf file requires two environment variables
   # PREFIX - the certificate prefix. This is used to decorate the distinguished name.
   # DN - this must be "CA" for root certs and "Signed" for signed certs.

   let N=$(cat ca-serial-no.txt) # signed certificate serial number.

   # Generate key for the signed cert.
   name="vgl-signed-${host}"
   key_file="${name}.key"
   cert_file="${name}.pem"

   rm -f ${key_file}
   PREFIX=vgl DN=Signed openssl genrsa -passout pass:claudio -out ${key_file} 2048

   # Generate the CSR for the signed cert.
   rm -f tmp.csr
   PREFIX=vgl DN=Signed openssl req -passin pass:claudio -config ./openssl.cnf -new -key ${key_file} -addext "subjectAltName=DNS:${host}" -batch -out tmp.csr

   # Generate the signed certificate. This is ugly but OpenSSL is broken - extensions (such as SANS) are flat out
   # not carried over from the CSR to the certificate. It's listed as a bug, but no indication it will ever get fixed.
   # Of course, there's no nice --addext option for certificate generation, so it's necessary to create a file
   # for the extension data.
   rm -f ${cert_file}
   cat > sans.tmp <<SANS
   [req_ext]
   subjectAltName = @alt_names

   [alt_names]
   # The hostname.
   DNS.1 = ${host}
   # The FQDN.
   DNS.2 = ${host}.solidwallofcode.com
   SANS

   # Finally the certificate can be generated.
   PREFIX=vgl DN=Signed openssl x509 -passin pass:claudio -req -in tmp.csr -CA vgl-ca.pem -CAkey vgl-ca.key \
   -set_serial 0$N -out ${cert_file} -days 36500 -sha256 -extfile sans.tmp -extensions req_ext

   # cleanup.
   rm -f tmp.csr sans.tmp
   let N=N+1
   echo ${N} > ca-serial-no.txt

Note - install a certificate for use requires both the certificate file ".pem" and the key file ".key".
The root certificate may need to be installed but the root key must *never* be deployed on a production
machine. It is used only to generate the signed certificates that are deployed.

Common Issues
*************

*  If the Portal page comes up completely blank, this likely due to "traffic_portal" not being able
   to connect to "traffic_ops". Check that
   *  "traffic_ops" host has port 443 open
   *  The name of the "traffic_ops" host resolved correctly on the "traffic portal" host.
   *  Check that "/etc/traffic_portal/conf/config.js" has been updated to have the "traffic ops" host.

*  If the initial admin login doesn't work, verify the password tweaks have been done. If not,
   it will be necessary to make then changes and then reset the password for the "traffic_ops"
   user. This is most easily done by logging in to the Postgresql host with this command.

   .. code-block::

      psql --host 127.0.0.1 --user traffic_ops

   There should be a password prompt - use the password specified in the "traffic ops" post install.

   .. code-block::

      \password

   Enter the password (twice) when prompted. Then exit Postgresql with the following command.

   .. code-block::

      \q

*  TLS verification requires a match between the hostname in the URL and the certificate. I solved
   this by putting the host name (base and FQDN) in the SAN data. A new certificate generation script
   was required. If not used, the following problem occurs.

   .. code-block::

      Login error Post "https://atc-ops/api/3.1/user/login": x509: certificate is not valid for any names, but wanted to match atc-ops

   This can be worked around by adding the option ``--traffic-ops-insecure=true`` to the ``t3c``
   command line but it's better to have correct certificates.

Issues
******

*  To do monitoring, the "astats_over_http" plugin must be used, but this is not in the ATS 9
   distribution and must be compiled separately.

To Do
*****

*  Recreate the Traffic Server VMs with a disk layout that supports raw device caching, then adjust
   the Traffic Ops Parameters to use those raw devices.

Footnotes
*********

.. [#]

   The Traffic Vault installation logic uses an SQL dump which requires the user name "traffic_vault".
   If that user doesn't exist then no tables are created. Moreover if it does exist it will be the
   owner of all of the tables. The user can be changed manually in the file and then the tables
   created using

   .. code-block:

      su -c 'psql traffic_vault < /opt/traffic_ops/app/db/trafficvault/create_tables.sql' - postgres

   The first argument is the database name (which is normally "traffic_vault"). The input seems to
   require the full path (likely due to ``su`` changing to the "postgres" user home directory).
   I recommend making a copy of the dump file, changing that, and then using the copy to initialize
   the database.

