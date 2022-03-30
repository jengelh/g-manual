.. sectnum::

Introduction
============

This document shall show, in a 27-step process, how to install Grommunio
components manually. It is targeted at adept administrators. Pictured above and
subsequently is the typical boot phase of many a contemporary Linux systems.

Pictures in this document depict a (what is initially) a fairly minimalist
openSUSE Tumbleweed container as a base. However, the choice of system used for
the screenshots shall be of no concern. The reader ought to be familiar with
any peculiarities that their particular environment has bestowed upon them, be
it in boot, in package management or configuration of services.

.. image:: login1.png

. . . Too complicated? Then use the grommunio Appliance.


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

* Frontend HTTP server (possibly as load balancers), e.g. nginx on port 443

  * Proxies to 9001 for URIs belonging to grommunio-web

  * Proxies to 10443 for URIs belonging to OXDISCO, MAPI, RPC

  * Proxies to an uwsgi for URIs belonging to admin-api

  * Serves up flat files (e.g. for grommunio-web/chat/meet/admin-web)

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

* grommunio-chat

  * Connects to MariaDB and optionally LDAP for authentication.

  * Mattermost application server with HTTP on port 8065.

* grommunio-meet

  * Connects to MariaDB and optionally LDAP for authentication.

  * Prosody application server with XMPP on port 5280.


Establish networking
====================

.. image:: network1.png

For this particular container, I had enabled ``systemd-networkd`` and put the
network configuration in place apriori. If anything, this section is but a
reminder to hook up the host to Internet, as it will be needed to get at
package repositories later. The particular method of network configuration
varies wildly between operating systems, and not every system is using
systemd-networkd. Consult the documentation relevant for your environment to
get online.

.. image:: network2.png

IPv6 is mandatory on the host itself. If you have ``::1`` assigned, all is
good.

You are well advised to install and configure a packet filter, a.k.a. a
firewall, with the sensible default of disallowing every service by default,
save perhaps for a way to let yourself in. More details will be presented
throughout the sections going forward.


Declare hostname identity
=========================

.. image:: hostname1.png

If you have not consciously set a hostname yet, do so now, especially if some
default setting has left you with localhost as the hostname. You cannot
reasonably reach localhost from another machine without unnecessary pains.

I decided to use ``route27.test`` for the domain part of later e-mail addresses
(e.g. ``someuser@route27.test``), and this particular machine that Grommunio
will be installed on has received a hostname of ``mail.route27.test``.
Arbitrary names can be chosen so long as they make sense for their intended
network.


Package manager setup
=====================

Visit `<https://download.grommunio.com>`_ to get an idea of the list of platforms for
which pre-built packages have been made available. Even though different
operating systems may use the same archive format (RPM, DEB, etc.) or
repository metadata formats (rpm-md, apt), do not use a repository which does
not exactly match your system. Do not use Debian packages for an Ubuntu system
or vice-versa. Do not use openSUSE packages for a Fedora system or vice-versa.
Do not even remotely think of converting between formats.

zypp
----

openSUSE uses yum-style ``.repo`` files for declaring repositories. Based on
the Tumbleweed container introduced earlier, one can create a file
``/etc/zypp/repos.d/grommunio.repo`` and populate it like so:

.. image:: repo-1.png

.. code-block::

	[grommunio]
	enabled=1
	autorefresh=1
	baseurl=https://download.grommunio.com/community/openSUSE_Tumbleweed
	type=rpm-md
	keeppackages=0

Retrieve the GPG key and import it into the RPM database to trust it. Then,
optionally, download the repository metadata (if not, it will be done the next
time you install anything).

.. image:: repo-2.png

.. image:: repo-3.png

.. code-block::

	curl https://download.grommunio.com/RPM-GPG-KEY-grommunio >gr.key
	rpm --import gr.key

dnf
---

RHEL uses ``.repo`` files as well, though in another directory. The file to edit
would be ``/etc/yum.repos.d/grommunio.repo``, with contents:

.. code-block::

	[grommunio]
	enabled=1
	autorefresh=1
	baseurl=https://download.grommunio.com/community/EL_8
	type=rpm-md
	keeppackages=0

Import the GPG key likewise, then proceed to use dnf or yum commands to update
at your leisure.

apt
---

For Debian, one is to add into ``/etc/apt/sources.list.d/grommunio.list``:

.. code-block::

	deb [trusted=yes] https://download.grommunio.com/community/Debian_11 Debian_11 main

Then import the GPG key and proceed to use apt commands to update at your
leisure.

.. image:: repodeb-1.png

.. image:: repodeb-2.png

.. image:: repodeb-3.png

For Ubuntu installations, the ``universe`` repository is required in addition
to the base install.


TLS certificates
================

For obtaining a certificate, refer to external documentation.

* Self-signed certificate: https://stackoverflow.com/a/10176685
* Using Let's Encrypt: https://certbot.eff.org/instructions

The certificate's key strictly needs to be passwordless, as most services have
no way to interactively ask for a password (they are launched in the background
anyway).

A certificate with a *subjectAltName* (SAN) field, or even a wildcard
certificate may be desirable for the domain, if you plan on using multiple
subdomains, e.g. ``meet.route27.test`` for *grommunio-meet*.

Autodiscover clients, as part of their setup attempts, try to resolve and use
``autodiscover.route27.test``. Having a SAN for this subdomain is however not
strictly necessary; we can report that Autodiscover also works without this
domain. See `MS-OXDISCO §3.1.5
<https://docs.microsoft.com/en-us/openspecs/exchange_server_protocols/ms-oxdisco/d56ae3c6-bf29-4712-b274-2e4cc5fdaa64>`_
about all the ways.

Advance list about which entities will prospectively need access to the
certificate(s):

* gromox

* nginx

* postfix (optional)

Some of the processes may read TLS certificates and their keyfiles *after*
switching to an unprivileged user identity. As a result, these files may need
to be enhanced with a filesystem ACL or, failing that, duplicate copies be made
with suitable ownership.


nginx
=====

nginx is used as a frontend to handle all HTTP requests, and to forward them to
further individual services. For example, RPC/HTTP requests will be delegated
to Gromox for further processing, Administration API (AAPI for short) requests
will be delegated to an uwsgi instance for further processing, and Mattermost
requests to the chat API.

An alternative HTTP server may be used if you feel comfortable in configuring
*all* of it, however this guide will only focus on nginx. Now then, source the
nginx package from your operating system, and have the service started both on
next boot and immediately.

.. image:: nginx-1.png

.. image:: nginx-2.png

In this screenshot, we also requested the installation of the nginx VTS module,
which AAPI can *optionally* for reporting traffic statistics. VTS is
**not** available for all platforms, in which case you have to omit and make do
without it.

Being the main entrypoint for everything, the nginx HTTPS network service,
generally port 443, will need to be configured in the packet filter to be
accessible (publicly).


nginx support package
=====================

We have a package that contains the first set of premade configuration
fragments for nginx. Do install the ``grommunio-common`` package.

.. code-block:: sh

	zypper in grommunio-common

The nginx default configuration as shipped by Linux distributions (file
``/etc/nginx/nginx.conf``) contains a line ``include conf.d/*``. The support
package places a file to ``/etc/nginx/conf.d/grommunio.conf``, such that the
nginx-related grommunio configuration gets automatically loaded on the next
nginx (re-)start.

The actual fragment files for nginx are located under
``/usr/share/grommunio-common`` for packaging policy reasons; they are not
meant to be modified. They do however has further ``include`` directives
pointing back to ``/etc`` to facilitate overriding specific aspects.

``/usr/share/grommunio-common/nginx/locations.d/autodiscover.conf`` for example
contains the fragment that tells nginx to recognize the ``/Autodiscover`` space
and forward such requests to gromox-http on port 10443 (see later section).


TLS for nginx
=============

Create ``/etc/grommunio-common/nginx/ssl_certificate.conf`` and populate with
the certificate directives, exchanging paths as appropriate:

.. code-block::

	ssl_certificate zzz.pem;
	ssl_certificate_key zzz.key;

(The exact chain of includes is ``/etc/nginx/nginx.conf`` ►
``/etc/nginx/conf.d/grommunio.conf`` ►
``/usr/share/grommunio-common/nginx.conf`` ►
``/etc/grommunio-common/nginx/ssl_certificate.conf``.)

The port 80 and 443 listen declarations are provided by
``/usr/share/grommunio-common/nginx.conf``.

nginx's configuration can be tested and shown, respectively:

.. code-block:: sh

	nginx -t
	nginx -T


MariaDB
=======

MariaDB/MySQL is used to store the user database amongst a few auxiliary
configuration parameters. If you plan on erecting a multi-host Gromox cluster,
this database is the one that is meant to be globally available to all nodes
that will eventually be running Gromox services.

A preexisting MariaDB server may be used. All the standard tools and
procedures that the world community has developed around SQL are applicable, in
terms of e.g. configuration, backup/restore, and replication.

Assuming though that you are going for a new SQL server instance, source the
MariaDB packages from your operating system, and have the service started
both on next boot and immediately.

.. image:: mysql-1.png

.. image:: mysql-2.png

After the installation, do create a blank database and user identity for
accessing it.

.. image:: mysql-3.png

.. code-block:: sql

	CREATE DATABASE `grommunio`;
	GRANT ALL ON `grommunio`.* TO 'grommunio'@'localhost' IDENTIFIED BY 'freddledgruntbuggly';

The MariaDB network service is not meant to be open to the public Internet.
Within your private network, it may need to be opened if (and only if) you plan
on using it in a multi-host Grommunio setup, or when your plans about database
replication demand it.

In certain versions, such as MySQL 8 (on e.g. Ubuntu 20.04), the GRANT
statement no longer implicitly creates users and one must use `CREATE USER
<https://dev.mysql.com/doc/refman/8.0/en/create-user.html>`_ instead.
Furthermore, authentication with MariaDB/older MySQL clients may fail due to
what appears to be a hashing method change; the remedy is an extra parameter
for CREATE USER or `ALTER USER
<https://stackoverflow.com/questions/49194719/>`_.


Gromox in general
=================

Gromox is the central groupware server component of grommunio. It provides
the services for Outlook RPC, IMAP/POP3, an LDA for ingestion, and a PHP
module for Z-MAPI.

The package is available by way of the Grommunio repositories. This guide is
subsequently based on such a pre-built Gromox. Experts wishing to build from
source and who have general knowledge on how to do so are referred to the
`Gromox installation documentation
<https://github.com/grommunio/gromox/doc/install.rst>`_ on specific aspects of
the build procedure.

.. image:: gromox-1.png

Gromox runs a number of processes and network services. None of them are meant
to be open to the public Internet, because nginx is already that important
point of ingress. The Gromox exmdb service (port 5000/tcp by default) needs to
be reachable from other Gromox nodes in a multi-host grommunio setup for
reasons of internal forwarding to a mailbox's home server.

Daemon executables are located in ``/usr/libexec/gromox``, they have short
names like ``http``, ``zcore``, etc. The manpage carries the same name, so you
would use ``man http`` to call up the corresponding manpage. The configuration
files read by default follow the same scheme, e.g. ``/etc/gromox/http.cfg``.
Process infomration utilities such as ps(1) may show the full path of the
executable or just ``http``, depending on how these diagnostic utilities are
used. The systemd unit name, though, is ``gromox-http.service``.

All log output goes to stderr. When run from systemd, this is automatically
redirected to the journal.


Gromox user database
====================

The connection parameters for MariaDB need to be conveyed to Gromox with the
file ``/etc/gromox/mysql_adaptor.cfg``, whose contents could look like this::

	mysql_username=grommunio
	mysql_password=freddledgruntbuggly
	mysql_dbname=grommunio
	schema_upgrade=host:mail.route27.test

The data stored in MariaDB is shared among all mailbox nodes in a clustered
setup. Table schema (DDL) changes are necessary at times, but at most one node
in such a cluster should perform these changes to avoid running the risk of
corruption. The hostname after ``host:`` specifies which machine will be
considered authoritative, if any. The ``schema_upgrade=host:...`` line should
be consistent across all mailbox nodes. It is possible to completely omit
``schema_upgrade``, at which point no updates will be done automatically.

With Gromox instrumented on the SQL parameters, proceed now with performing the
initial creation of the database tables by issuing the command:

.. code-block:: sh

	gromox-dbop -C

.. image:: gromox-2.png

If automatic schema upgrades are disabled, manual updates can be performed
later with:

.. code-block:: sh

	gromox-dbop -U


gromox-event/timer
==================

* event: A notification daemon for an interprocess channel between
gromox-imap/gromox-midb. No configuration needed.
* timer: An at(1)/atd(8)-like daemon for delayed delivery. No configuration
needed.

.. code-block:: sh

	systemctl enable --now gromox-event gromox-timer


gromox-http
===========

Because nginx was set up earlier as a frontend to listen on ports 80 and 443,
gromox-http needs to be moved "out of the way" (its built-in defaults are also
80/443). In addition, the daemon needs to be told the paths to the TLS
certificates. A manual page is provided with all the configuration directives
and can be called up with ``man 8gx http``. For now, these directives should
suffice:

.. code-block::

	listen_port=10080
	listen_ssl_port=10443
	http_support_ssl=yes
	http_certificate_path=zzz.pem
	http_private_key_path=zzz.key

Run the service.

.. code-block:: sh

	systemctl enable --now gromox-http

Perform a connection test. The expected result of requesting the ``/`` URI will
be a 404 status code. (It could serve a static HTML file, but the default
config has no such file, and ``/`` is not mapped anywhere. Mayb we should
change that…)

.. code-block:: sh

	curl -kv https://localhost:10443/

.. code-block::

	> GET / HTTP/1.1
	> Host: localhost:10443
	…
	< HTTP/1.1 404 Not Found
	…

Gromox's default config does however has a mapping for ``/web`` (to
``/usr/share/grommunio-web``). If you happen have the ``grommunio-web`` package
already installed, requests to this subdirectory can be responded to. You can
test the following URLs (port 10443 for gromox-http directly, 443 for nginx,
respectively) that should service a static file:

.. code-block:: sh

	curl -kv https://localhost:10443/web/robots.txt
	curl -kv https://localhost:443/web/robots.txt

.. code-block::

	< HTTP/1.1 200 OK
	< Date: Tue, 29 Mar 2022 23:08:33 GMT
	< Content-Type: text/plain
	< Content-Length: 26
	< Accept-Ranges: bytes
	< Last-Modified: Tue, 29 Mar 2022 07:09:12 GMT
	< ETag: "19165e1100000000-1a000000-98b0426200000000"
	<
	User-agent: *
	Disallow: /


gromox-midb/zcore
=================

The IMAP Message Index Database, and the bridge process for PHP-MAPI. No
further configuration needed.

.. code-block:: sh

	systemctl enable --now gromox-midb gromox-zcore


gromox-imap/pop3
================

Similar to ``http.cfg``, convey to the IMAP/POP3 daemons the TLS certificate
paths. Skip this section if you do not intend to run these protocols.

IMAP/POP3 can run in unencrypted mode, but only for developers. Hence,
imap_force_starttls is set here. In ``/etc/gromox/imap.cfg``, declare:

.. code-block::

	listen_ssl_port=993
	imap_support_starttls=true
	imap_certificate_path=zzz.pem
	imap_private_key_path=zzz.key
	imap_force_starttls=true

In ``/etc/gromox/pop3.cfg``:

.. code-block::

	listen_ssl_port=995
	pop3_support_stls=true
	pop3_certificate_path=zzz.pem
	pop3_private_key_path=zzz.key
	pop3_force_stls=true

Enable/start zero or more of the services you wish to utilize:

.. code-block:: sh

	systemctl enable --now gromox-imap gromox-pop3


PHP-FPM
=======

The installation of the ``gromox`` package should have already pulled in
php-fpm as a dependency.

For completeness, verify that PHP knows about the MAPI module.

.. code-block:: sh

	echo -en '<?php phpinfo(); ?>' | php | grep mapi

Verify that the gromox pool file was placed.

.. code-block:: sh

	ls -al /etc/php8/fpm/php-fpm.d/gromox.conf

Then enable/start php-fpm:

.. code-block:: sh

	systemctl enable --now php-fpm

For completness, verify that the socket in the pool file was created:

.. code-block:: sh

	ls -al /run/gromox/php-fpm.sock

Try to elicit a response from the Autodiscover code, via gromox-http (10443)
and/or nginx (443).
(``/usr/share/grommunio-common/nginx/locations.d/autodiscover.conf`` defines
the handler for the ``/Autodiscover`` URI path, to pass all requests to
gromox-http on port 10443. gromox-http forwards this to php-fpm. This way,
Autodiscover also works in test setups without a frontend like nginx.)

.. code-block:: sh

	curl -kv https://localhost:10443/Autodiscover/Autodiscover.xml
	curl -kv https://localhost:443/Autodiscover/Autodiscover.xml

Expected result of this operation:

.. code-block::

	> GET /Autodiscover/Autodiscover.xml HTTP/1.1
	> Host: localhost:10443
	…
	< HTTP/1.1 200 Success
	< Date: Tue, 29 Mar 2022 23:54:16 GMT
	< Transfer-Encoding: chunked
	< Content-type: text/html; charset=UTF-8
	<
	E-2000: invalid request method, must be POST!


Administration API (AAPI)
=========================

Install ``grommunio-admin-api``.

.. code-block::

	zypper in grommunio-admin-api

Fragments are placed in /usr/share/grommunio-common/...
In the nginx configuration, include this fragment ...


Permissions
-----------

AAPI writes to system configuration files. As far as Gromox is concerned,
``/etc/gromox`` should have owner ``root:gromox`` and mode 0771 or 0775. The
filelist is known in advance, so there is little gain in removing the ``o+r``
bit.


Administration Web Interface (AWEB)
===================================


* Create user
* Authentication, Autodiscover test
* Outlook connection test


grommunio-web
=============

Install ``grommunio-web``. Verify that you can load the login page. This is
reachable by both gromox-http (10443) and nginx (443):

.. code-block:: sh

	curl -kv https://localhost:443/web/


Loopback mail
=============


Postfix
=======


Gromox multiserver
==================

On each node, ``/etc/gromox/exmdb_acl.txt`` needs to be placed and contain a
list of IP addresses of all the mailbox nodes.
