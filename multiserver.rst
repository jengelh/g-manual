Scale out considerations
========================

To deploy grommunio over multiple hosts/containers/etc., each host needs to be
populated with a base operating system and the grommunio packages (all, or a
subset).

In case of deployment to a dedicated host (bare metal or virtual machine), the
base OS and packages can be provided by the ISO/OVA appliance images. The
filesystem tree from the RAW image can be used for container environments
(boot-from-directory). Equally though, you can install a base OS of choice
yourself and fill it with the grommunio packages.

YOU decide how you want to split the required resources (CPUs, RAM, disk)
across hosts. You can put all mailboxes and all services on a single monster
machine, or you can create one container for every individual mailbox and every
individual user-facing service. Both extremes are possible, though not
necessarily cost-effective. One big iron machine may cost more to procure than
two irons of half the size. Likewise, running two mailboxes rather than one per
container is more effective because the operating system and processes need not
be replicated that often. However you split it is a determination YOU must
make.


Components
==========

Services that can be placed on different nodes:

* LDAP/IDM

* MariaDB/MySQL for user metadata

* Mailbox nodes

  * gromox-http/exmdb_provider:

    * Provides a trivial API on port 5000 to serialize SQLite access.

    * Only little state (state which is visible to all consuming service
      at the same time), e.g. search folders.

  * gromox-http:

    * Application server with HTTP on port 10443.

    * Performs user authentication. Connects to MariaDB and optionally LDAP.

    * Provides the handler for AutoDiscover URIs.
      Connects to exmdb to read the mailbox.

    * Provides the handler for MAPI URIs.
      Connects to exmdb to read the mailbox.

    * Keeps the Outlook session state, e.g. object handles.

* PHP nodes

  * gromox-zcore

    * Provides a trivial API on an AF_LOCAL socket

    * Performs user authentication. Connects to MariaDB and optionally LDAP.

    * Keeps the grommunio-web session state, e.g. --

    * Connects to exmdb to read the mailbox.

  * php-fpm: provides a FastCGI API on port 9001

    * runs grommunio-web PHP code

    * connects to gromox-zcore for state and mailbox access

* Frontend HTTP server (possibly as a fedration of load balancers later down
  the road), e.g. nginx, on port 443

  * Proxies to 9001 for URIs belonging to grommunio-web

  * Proxies to 10443 for URIs belonging to OXDISCO, MAPI, RPC

  * Serves up flat files (e.g. for grommunio-web/chat/meet/admin-web)

  * Also listens on 8080/8443 and serves the Administrator interface there
    (more flat files).

  * Proxies to an AF_LOCAL socket for URIs belonging to the AAPI REST
    interface.

* gromox-midb

  * Caches mailbox data to speed up IMAP access.

  * Provides a trivial API on port 5000 for this IMAP-level meta data

  * Connects to exmdb to read/write the mailbox.

* gromox-imap

  * Performs user authentication. Connects to MariaDB and optionally LDAP.

  * Provides IMAP on port 143.

  * Connects to midb to read/write the mailbox.

* gromox-pop3

  * Performs user authentication. Connects to MariaDB and optionally LDAP.

  * Provides IMAP on port 143.

  * Connects to midb to read/write the mailbox.

* gromox-delivery

  * Mail ingester on port 24.

  * Connects to midb to write metadata.

  * Connects to exmdb to write mailbox.

* postfix or other MTA

  * Optionally performs user authentication (e.g. outgoing mail).
    Connects to MariaDB and optionally LDAP.

  * Provides SMTP on port 25, optionally 587.

  * Connects to gromox-delivery for mail ingesting.

  * [[ rspamd may run on port 11334 ]]

* grommunio-chat

  * Connects to MariaDB and optionally LDAP for authentication.

  * Mattermost application server with HTTP on port 8065.

* grommunio-meet

  * Connects to MariaDB and optionally LDAP for authentication.

  * Prosody application server with XMPP on port 5280.

* Administration interface; REST interface and HTML/JS frontend


Gromox multiserver
==================

On each node, ``/etc/gromox/exmdb_acl.txt`` needs to be placed and contain a
list of IP addresses of all the mailbox nodes.
