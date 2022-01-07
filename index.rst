0. Introduction
===============

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


1. Establish networking
=======================

.. image:: network1.png

For this particular container, I had enabled ``systemd-networkd`` and put the
network configuration in place apriori. If anything, this section is but a
reminder to hook up the host to Internet, as it will be needed to get at
package repositories later. The particular method of network configuration
varies wildly between Linux distributions, and not every system is using
systemd-networkd. Consult the documentation relevant for your environment to
get online.

.. image:: network2.png

IPv6 is mandatory on the host itself. If you have ``::1`` assigned, all is
good.


2. Declare hostname identity
============================

.. image:: hostname1.png

If you have not consciously set a hostname yet, do so now, especially if some
default setting has left you with localhost as the hostname. You cannot
reasonably reach localhost from another machine without unnecessary pains.

I decided to use ``route27.test`` for the domain part of later e-mail addresses
(e.g. ``someuser@route27.test``), and this particular machine that Grommunio
will be installed on has received a hostname of ``mail.route27.test``.
Arbitrary names can be chosen so long as they make sense for their intended
network.


3. Package manager setup
========================

Visit ``download.grommunio.com`` to get an idea of the list of platforms for
which pre-built packages have been made available. Even though different
distribution projects may use the same archive format (RPM, DEB, etc.) or
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


4. NGINX
========

Install ``nginx``. (...)


5. MariaDB
==========

Install MariaDB/MySQL and have the service started both on next boot and
immediately, then create a blank database and a user that can access it.

.. image:: mysql-1.png

.. image:: mysql-2.png

Following that, create a blank database and user identity for accessing it.

.. image:: mysql-3.png

.. code-block:: sql

	CREATE DATABASE `grommunio`;
	GRANT ALL ON `grommunio`.* TO 'grommunio'@'localhost' IDENTIFIED BY 'freddledgruntbuggly';


6. Gromox
=========

Install the package:

.. image:: gromox-1.png

Then specify the MariaDB connection parameters by placing them in
``/etc/gromox/mysql_adaptor.cfg``:

.. code-block::

	mysql_username=grommunio
	mysql_password=freddledgruntbuggly
	mysql_dbname=grommunio

Subsequently, perform the initial table creation:

.. code-block::

	gromox-dbop -C

.. image:: gromox-2.png


7. Administration interface
===========================

Install ``grommunio-admin-api``

.. code-block::

	zypper in grommunio-admin-api grommunio-admin-web

Fragments are placed in /usr/share/grommunio-common/...
In the nginx configuration, include this fragment ...


8. Create first user
====================

...


9. Outlook connection
=====================

...


10. grommunio-web frontend
==========================

.. code-block::

	zypper in grommunio-web

(hook up to nginx / gromox pool / ...)
