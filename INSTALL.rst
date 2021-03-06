This document describes the installation and setup of HaaS on CentOS 7.0.
HaaS should work on other distros, but is not well tested or supported.
For development environments, see ``HACKING.rst``.

The HaaS Service node
=====================

This section talks about what must be done on the server upon which HaaS runs.

Prerequisite Software
---------------------

HaaS requires a number of packages available through the CentOS RPM
repositories, as well as the EPEL repository. EPEL can be enabled via::

    yum install epel-release

Then, the rest of the packages can be installed via::

    yum install libvirt bridge-utils ipmitool telnet httpd mod_wsgi python-pip qemu-kvm python-virtinst virt-install python-psycopg2

In addition, HaaS depends on a number of python libraries. Many of these are
available as RPMs as well, but we recommend installing them with pip, since
this will install the versions that HaaS has been tested with.  This is done
automatically by the instructions below.

Setting Up HaaS Database
------------------------

The only DBMS currently supported for production use is PostgreSQL. (SQLite is
supported for development purposes *only*). There are many guides on the web
which describe setting up PostgreSQL; in these instructions we assume that
you've done so successfully.

You need to create a postgresql user and database for HaaS. Once you've done so,
the ``uri`` option in ``haas.cfg``'s ``database`` must be set to::

        postgresql://<user>:<password>@<address>/<dbname>

Where ``<user>`` is the name of the postgres user you created, ``<password>`` is
its password, ``<dbname>`` is the name of the database you created, and
``<address>`` is the address which haas should use to connect to postgres (In a
typical default postgres setup, the right value is ``localhost``).


The HaaS software itself can then be installed by running::

    git clone https://github.com/CCI-MOC/haas
    cd haas
    sudo python setup.py install

Disable SELinux
---------------

The setup described in this document runs into problems with SELinux. In the
future, we hope to ship a set of SELinux security policies with HaaS, but for
now the solution is to disable SELinux::

    sudo setenforce 0

Make sure SELinux is also disabled on startup.

Create User
-----------

First create a system user ``haas_user`` with::

  sudo useradd haas_user -d /var/lib/haas -m -r

haas.cfg
--------

HaaS is configured with ``haas.cfg``. This file contains settings for both the
CLI client and the server. Carefully read the ``haas.cfg*`` files in
``examples/``, to understand and correctly set all of the options.  In
particular, the following two fields in the ``headnode`` section are very
important: ``trunk_nic`` must match your choice of trunk NIC in the "Networking
- Bridges" instructions below; ``base_imgs`` must match the name of the base
  headnode libvirt instance created in the "Libvirt" instructions below.

For a detailed description of the configuration needed for various switch
setups, see ``docs/network-drivers.md``.

The file should be placed at ``/etc/haas.cfg``; The ``haas.wsgi``
script, described below, requires this. Awkwardly, the ``haas``
command line tool expects the file to be present in its current
working directory. This will be fixed in the next release, but for
now, put the file in ``/etc`` and create a symlink to it in the
HaaS user's home directory::

  sudo ln -s /etc/haas.cfg /var/lib/haas/

It should be noted that HaaS end users will also require a ``haas.cfg`` file
in their local directory in order to communicate with the HaaS server.
However, creating another symlink to the ``/etc/haas.cfg`` exposes sensitive
administrative information to users, such as usernames and passwords. To
avoid this, users should have their own copy of ``haas.cfg`` that is stripped
of all sections except for the ``[client]`` section.  Additionally, the
``/etc/haas.cfg`` should have its permissions set to read-only and ownership
set to the ``haas_user``::

  sudo chown haas_user:haas_user /etc/haas.cfg
  sudo chmod 400 /etc/haas.cfg

All HaaS commands in these instructions should be run in this directory::

  cd /var/lib/haas

Networking - Bridges
--------------------

Currently HaaS only supports one mechanism for layer-2 isolation: 802.1q VLANs.
One NIC on the HaaS host is designated the "trunk NIC".  All network traffic to
headnode VMs in HaaS is routed through this trunk NIC, on a tagged VLAN.  As
such, the port on the switch that this NIC connects to must have all of HaaS's
VLANs trunked to it.  Currently, this configuration must be done manually.

HaaS uses Linux bridges to route the traffic from the trunk NIC to the
headnodes. Currently the bridges and VLAN NICs for this must be created
ahead of time.  The provided script ``create_bridges`` will create bridges
for all VLANS in the allocation pool. It must be run in the directory that
contains ``haas.cfg``. This pre-allocation is easier to reason about
than on-demand creation, and allows HaaS to be run as an unprivileged user,
but it also causes some limitations.  For instance, because of this, headnodes
can only be connected to networks with allocated VLANs.  The bridges must also
be pre-allocated again on each boot. For now, the recomended approach is to add::

  (cd /etc && create_bridges)

to the end of ``/etc/rc.local``.

HaaS must additionally have IP connectivity to the switch's administration
console.  Right now the only mechanism for connecting to the switch is via
telnet (with `plans <https://github.com/CCI-MOC/haas/issues/46>`_ to support
ssh). As such, the administration console should only be accessible through a
trusted private network.

Libvirt
-------

We must reconfigure ``libvirt`` to allow (some) unprivileged users access to
the system QEMU session.  To do this, edit ``/etc/libvirt/libvirtd.conf`` and
uncomment the following lines::

  unix_sock_group = "libvirt"
  auth_unix_ro = "none"
  auth_unix_rw = "none"

Then create the group 'libvirt' and add the HaaS user to that group::

  sudo groupadd libvirt
  sudo gpasswd libvirt -a haas_user

Finally, restart ``libvirt`` with::

  sudo service libvirtd restart

You should also set libvirt to start on boot::

  sudo chkconfig libvirtd on

Headnode image
^^^^^^^^^^^^^^
Now we must make a clonable base headnode.  (One is required, and more are
allowed.)  First create a storage pool.  Any kind can be used, but we will only
document creating a directory-backed storage pool::

  virsh --connect qemu:///system pool-define pool.xml

where ``pool.xml`` contains a description of the pool::

  <pool type="dir">
    <name>haas_headnodes</name>
    <target>
      <path>/var/lib/libvirt/images</path>
    </target>
  </pool>

The directory specified by path must already exist, and be readable and
writable by the ``libvirt`` user. Then activate the pool, and make the it
activate on boot, with::

  virsh --connect qemu:///system pool-start haas_headnodes
  virsh --connect qemu:///system pool-autostart haas_headnodes

The scripts in ``examples/cloud-img-with-passwd`` can be used to build
an ubuntu 14.04 or centos 7 disk image with a default root password. Read
the README in that directory for more information.

Once the disk image is built, copy ito the storage pool directory (here we
assume it is called ``base.img``)::

  mv base.img /var/lib/libvirt/images/

Finally, create the base headnode with::

  virsh --connect qemu:///system define base.xml

where ``base.xml`` contains a description of the headnode::

  <domain type='kvm'>
    <name>base</name>
    <memory>524288</memory>
    <os>
      <type arch='x86_64'>hvm</type>
      <boot dev='hd'/>
    </os>
    <features>
      <acpi/><apic/><pae/>
    </features>
    <clock offset="utc"/>
    <on_poweroff>destroy</on_poweroff>
    <on_reboot>restart</on_reboot>
    <on_crash>restart</on_crash>
    <vcpu>1</vcpu>
    <devices>
      <emulator>/usr/libexec/qemu-kvm</emulator>
      <disk type='file' device='disk'>
        <driver name='qemu' type='raw'/>
        <source file='/var/lib/libvirt/images/base.img'/>
        <target dev='vda' bus='virtio'/>
      </disk>
      <interface type='network'>
        <source network='default'/>
        <model type='virtio'/>
      </interface>
      <input type='tablet' bus='usb'/>
      <graphics type='vnc'/>
      <console type='pty'/>
      <sound model='ac97'/>
      <video>
        <model type='cirrus'/>
      </video>
    </devices>
  </domain>

Note that the above specifies the format of the disk image as ``raw``; if
you're using an image in another format (such as ``qcow``) you will have
to adjust this.

Many of these fields are probably not needed, but we have not thouroughly
tested which ones. Furthermore, this set of XML duplicates the path to
storage directory; this seems unnecessary.

Users may find the scripts in ``examples/puppet_headnode`` useful for
configuring the ubuntu headnode to act as a PXE server; see the README in
that directory for more information.

Database
------------

HaaS currently supports SQLite for maintaining state. Because SQLAlchemy is
used as a database access layer, other DBs can and should be easily supported
in future releases. The database must be readable and writable by the HaaS
user.  Running the following command as ``haas_user`` will create it (in the
location specified in ``haas.cfg``) and initialize its tables::

  haas init_db


Authentication and Authorization
--------------------------------

HaaS includes a pluggable architecture for authentication and authorization.
HaaS ships with two authentication backends. One uses HTTP basic auth, with
usernames and passwords stored in the haas database. The other is a "null"
backend, which does no authentication or authorization checks. This can be
useful for testing and experimentation but *should not* be used in production.
You must enable exactly one auth backend.

Database Backend
^^^^^^^^^^^^^^^^

To enable the database backend, make sure the **[extensions]** section of
``haas.cfg`` contains::

  haas.ext.auth.database =

Then initialize the database as described above. You will need to add an
initial user with administrative privileges to the database in order to
bootstrap the system. You can do this by running the following command from
within the directory containing the server's ``haas.cfg``::

  haas create_admin_user <username> <password>

You can then create additional users via the HTTP API. You may want to
subsequently delete the initial user; this can also be done via the API.

Null Backend
^^^^^^^^^^^^

To enable the null backend, make sure **[extensions]** contains::

  haas.ext.auth.null =


Running the Server under Apache
-------------------------------

HaaS consists of two services: an API server and a networking server. The
former is a WSGI application, which we recommend running with Apache's
``mod_wsgi``. Create a file ``/etc/httpd/conf.d/wsgi.conf``, with the contents::

  LoadModule wsgi_module modules/mod_wsgi.so
  WSGISocketPrefix run/wsgi

  <VirtualHost 127.0.0.1:80>
    ServerName 127.0.0.1
    AllowEncodedSlashes On
    WSGIDaemonProcess haas_user user=haas_user group=haas_user threads=2
    WSGIScriptAlias / /var/www/haas/haas.wsgi
    <Directory /var/www/haas>
      WSGIProcessGroup haas_user
      WSGIApplicationGroup %{GLOBAL}
      Order deny,allow
      Allow from all
    </Directory>
  </VirtualHost>

(The file may already exist, with just the ``LoadModule`` option. If so, it is
safe to replace it.)

**Note:** certain calls to HaaS such as *port_register()* may pass arbitrary
strings that should be escaped (see `issue 361 <https://github.com/CCI-MOC/haas/issues/360>`_). By default, Apache `Doesn't
allow <https://stackoverflow.com/questions/4390436/need-to-allow-encoded-slashes-on-apache>`_
this due to security concerns. ``AllowEncodedSlashes On`` enables the passing
of these arguments.

If you haven't already, create the directory that will contain the HaaS WSGI module::

 sudo mkdir /var/www/haas/

Copy the file ``haas.wsgi`` from the top of the haas source tree to the
location indicated by the ``WSGIScriptAlias`` option. The virtual host and
server name should be set according to the hostname (and port) by which clients
will access the api. Then, restart Apache::

  sudo service httpd restart

You should also set apache to start on boot::

  sudo chkconfig httpd on

Running the network server:
---------------------------

Using systemd:
--------------

A systemd script for running the network server is available in the 'scripts' directory.
Name of the script is: haas_network.service

Centos:
-------

Centos uses systemd to controll all its processes. 

Place the file haas_network.service under:
``/usr/lib/systemd/system/``

Ubuntu:
-------
Systemd is available from Ubuntu 15.04 onwards and LTS version 16.04 will ship with systemd by default.

Place the file haas_network.service under:
``/lib/systemd/system/``


Starting the service:
---------------------

Following commands will start the daemon:
``systemctl daemon-reload``
``systemctl start haas_network``

You can check the status using:
``systemctl status haas_network``

To auto-start the service on boot:
``systemctl enable haas_network``


For systems that do not support systemd:
----------------------------------------
Some systems like the LTS version of Ubuntu, Ubuntu 14.04 does not come with systemd pre-installed.
It uses "Upstart" an equivalent of systemd to manage its daemons/processes.

For such systems, the networking server may be started as the HaaS user by running::

  haas serve_networks &

To make this happen on boot, add the following to ``/etc/rc.local``::

  (cd /var/lib/haas && su haas_user -c 'haas serve_networks') &


Congratulations- at this point, you should have a functional HaaS service running!

Describe datacenter resources
===================================

For HaaS to do anything useful, you must use the HaaS API to populate the
database with information about the resources in your datacenter -- chiefly
nodes, their NICs and the ports to which those NICs are attached. These are
the relevant API calls:

- ``node_register``
- ``node_delete``
- ``node_register_nic``
- ``node_delete_nic``
- ``port_register``
- ``port_delete``
- ``port_connect_nic``
- ``port_detach_nic``

