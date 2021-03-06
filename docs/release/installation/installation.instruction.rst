.. This work is licensed under a Creative Commons Attribution 4.0 International License.
.. http://creativecommons.org/licenses/by/4.0
.. (c) Open Platform for NFV Project, Inc. and its contributors

========
Abstract
========

This document describes how to install the Fraser release of
OPNFV when using Fuel as a deployment tool, covering its usage,
limitations, dependencies and required system resources.
This is an unified documentation for both x86_64 and aarch64
architectures. All information is common for both architectures
except when explicitly stated.

============
Introduction
============

This document provides guidelines on how to install and
configure the Fraser release of OPNFV when using Fuel as a
deployment tool, including required software and hardware configurations.

Although the available installation options provide a high degree of
freedom in how the system is set up, including architecture, services
and features, etc., said permutations may not provide an OPNFV
compliant reference architecture. This document provides a
step-by-step guide that results in an OPNFV Fraser compliant
deployment.

The audience of this document is assumed to have good knowledge of
networking and Unix/Linux administration.

=======
Preface
=======

Before starting the installation of the Fraser release of
OPNFV, using Fuel as a deployment tool, some planning must be
done.

Preparations
============

Prior to installation, a number of deployment specific parameters must be collected, those are:

#.     Provider sub-net and gateway information

#.     Provider VLAN information

#.     Provider DNS addresses

#.     Provider NTP addresses

#.     Network overlay you plan to deploy (VLAN, VXLAN, FLAT)

#.     How many nodes and what roles you want to deploy (Controllers, Storage, Computes)

#.     Monitoring options you want to deploy (Ceilometer, Syslog, etc.).

#.     Other options not covered in the document are available in the links above


This information will be needed for the configuration procedures
provided in this document.

=========================================
Hardware Requirements for Virtual Deploys
=========================================

The following minimum hardware requirements must be met for the virtual
installation of Fraser using Fuel:

+----------------------------+--------------------------------------------------------+
| **HW Aspect**              | **Requirement**                                        |
|                            |                                                        |
+============================+========================================================+
| **1 Jumpserver**           | A physical node (also called Foundation Node) that     |
|                            | will host a Salt Master VM and each of the VM nodes in |
|                            | the virtual deploy                                     |
+----------------------------+--------------------------------------------------------+
| **CPU**                    | Minimum 1 socket with Virtualization support           |
+----------------------------+--------------------------------------------------------+
| **RAM**                    | Minimum 32GB/server (Depending on VNF work load)       |
+----------------------------+--------------------------------------------------------+
| **Disk**                   | Minimum 100GB (SSD or SCSI (15krpm) highly recommended)|
+----------------------------+--------------------------------------------------------+


===========================================
Hardware Requirements for Baremetal Deploys
===========================================

The following minimum hardware requirements must be met for the baremetal
installation of Fraser using Fuel:

+-------------------------+------------------------------------------------------+
| **HW Aspect**           | **Requirement**                                      |
|                         |                                                      |
+=========================+======================================================+
| **# of nodes**          | Minimum 5                                            |
|                         |                                                      |
|                         | - 3 KVM servers which will run all the controller    |
|                         |   services                                           |
|                         |                                                      |
|                         | - 2 Compute nodes                                    |
|                         |                                                      |
+-------------------------+------------------------------------------------------+
| **CPU**                 | Minimum 1 socket with Virtualization support         |
+-------------------------+------------------------------------------------------+
| **RAM**                 | Minimum 16GB/server (Depending on VNF work load)     |
+-------------------------+------------------------------------------------------+
| **Disk**                | Minimum 256GB 10kRPM spinning disks                  |
+-------------------------+------------------------------------------------------+
| **Networks**            | 4 VLANs (PUBLIC, MGMT, STORAGE, PRIVATE) - can be    |
|                         | a mix of tagged/native                               |
|                         |                                                      |
|                         | 1 Un-Tagged VLAN for PXE Boot - ADMIN Network        |
|                         |                                                      |
|                         | Note: These can be allocated to a single NIC -       |
|                         | or spread out over multiple NICs                     |
+-------------------------+------------------------------------------------------+
| **1 Jumpserver**        | A physical node (also called Foundation Node) that   |
|                         | hosts the Salt Master and MaaS VMs                   |
+-------------------------+------------------------------------------------------+
| **Power management**    | All targets need to have power management tools that |
|                         | allow rebooting the hardware and setting the boot    |
|                         | order (e.g. IPMI)                                    |
+-------------------------+------------------------------------------------------+

.. NOTE::

    All nodes including the Jumpserver must have the same architecture (either x86_64 or aarch64).

.. NOTE::

    For aarch64 deployments an UEFI compatible firmware with PXE support is needed (e.g. EDK2).

===============================
Help with Hardware Requirements
===============================

Calculate hardware requirements:

For information on compatible hardware types available for use,
please see `Fuel OpenStack Hardware Compatibility List <https://www.mirantis.com/software/hardware-compatibility/>`_

When choosing the hardware on which you will deploy your OpenStack
environment, you should think about:

- CPU -- Consider the number of virtual machines that you plan to deploy in your cloud environment and the CPUs per virtual machine.

- Memory -- Depends on the amount of RAM assigned per virtual machine and the controller node.

- Storage -- Depends on the local drive space per virtual machine, remote volumes that can be attached to a virtual machine, and object storage.

- Networking -- Depends on the Choose Network Topology, the network bandwidth per virtual machine, and network storage.

================================================
Top of the Rack (TOR) Configuration Requirements
================================================

The switching infrastructure provides connectivity for the OPNFV
infrastructure operations, tenant networks (East/West) and provider
connectivity (North/South); it also provides needed connectivity for
the Storage Area Network (SAN).
To avoid traffic congestion, it is strongly suggested that three
physically separated networks are used, that is: 1 physical network
for administration and control, one physical network for tenant private
and public networks, and one physical network for SAN.
The switching connectivity can (but does not need to) be fully redundant,
in such case it comprises a redundant 10GE switch pair for each of the
three physically separated networks.

The physical TOR switches are **not** automatically configured from
the Fuel OPNFV reference platform. All the networks involved in the OPNFV
infrastructure as well as the provider networks and the private tenant
VLANs needs to be manually configured.

Manual configuration of the Fraser hardware platform should
be carried out according to the `OPNFV Pharos Specification
<https://wiki.opnfv.org/display/pharos/Pharos+Specification>`_.

============================
OPNFV Software Prerequisites
============================

The Jumpserver node should be pre-provisioned with an operating system,
according to the Pharos specification. Relevant network bridges should
also be pre-configured (e.g. admin_br, mgmt_br, public_br).

- The admin bridge (admin_br) is mandatory for the baremetal nodes PXE booting during Fuel installation.
- The management bridge (mgmt_br) is required for testing suites (e.g. functest/yardstick), it is
  suggested to pre-configure it for debugging purposes.
- The public bridge (public_br) is also nice to have for debugging purposes, but not mandatory.

The user running the deploy script on the Jumpserver should belong to ``sudo`` and ``libvirt`` groups,
and have passwordless sudo access.

The following example adds the groups to the user ``jenkins``

.. code-block:: bash

    $ sudo usermod -aG sudo jenkins
    $ sudo usermod -aG libvirt jenkins
    $ reboot
    $ groups
    jenkins sudo libvirt

    $ sudo visudo
    ...
    %jenkins ALL=(ALL) NOPASSWD:ALL

The folder containing the temporary deploy artifacts (``/home/jenkins/tmpdir`` in the examples below)
needs to have mask 777 in order for libvirt to be able to use them.

.. code-block:: bash

    $ mkdir -p -m 777 /home/jenkins/tmpdir

For an AArch64 Jumpserver, the ``libvirt`` minimum required version is 3.x, 3.5 or newer highly recommended.
While not mandatory, upgrading the kernel and QEMU on the Jumpserver is also highly recommended
(especially on AArch64 Jumpservers).

For CentOS 7.4 (AArch64), distro provided packages are already new enough.
For Ubuntu 16.04 (arm64), distro packages are too old and 3rd party repositories should be used.
For convenience, Armband provides a DEB repository holding all the required packages.

To add and enable the Armband repository on an Ubuntu 16.04 system,
create a new sources list file ``/apt/sources.list.d/armband.list`` with the following contents:

.. code-block:: bash

    $ cat /etc/apt/sources.list.d/armband.list
    //for OpenStack Queens release
    deb http://linux.enea.com/mcp-repos/queens/xenial queens-armband main

    $ apt-get update

Fuel@OPNFV has been validated by CI using the following distributions
installed on the Jumpserver:

- CentOS 7 (recommended by Pharos specification);
- Ubuntu Xenial;

.. WARNING::

    The install script expects ``libvirt`` to be already running on the Jumpserver.
    In case ``libvirt`` packages are missing, the script will install them; but
    depending on the OS distribution, the user might have to start the ``libvirtd``
    service manually, then run the deploy script again. Therefore, it
    is recommended to install libvirt-bin explicitly on the Jumpserver before the deployment.

.. NOTE::

    It is also recommended to install the newer kernel on the Jumpserver before the deployment.

.. WARNING::

    The install script will automatically install the rest of required distro package
    dependencies on the Jumpserver, unless explicitly asked not to (via ``-P`` deploy arg).
    This includes Python, QEMU, libvirt etc.

.. WARNING::

    The install script will alter Jumpserver sysconf and disable ``net.bridge.bridge-nf-call``.

.. code-block:: bash

    $ apt-get install linux-image-generic-hwe-16.04-edge libvirt-bin


==========================================
OPNFV Software Installation and Deployment
==========================================

This section describes the process of installing all the components needed to
deploy the full OPNFV reference platform stack across a server cluster.

The installation is done with Mirantis Cloud Platform (MCP), which is based on
a reclass model. This model provides the formula inputs to Salt, to make the deploy
automatic based on deployment scenario.
The reclass model covers:

   - Infrastructure node definition: Salt Master node (cfg01) and MaaS node (mas01)
   - OpenStack node definition: Controller nodes (ctl01, ctl02, ctl03) and Compute nodes (cmp001, cmp002)
   - Infrastructure components to install (software packages, services etc.)
   - OpenStack components and services (rabbitmq, galera etc.), as well as all configuration for them


Automatic Installation of a Virtual POD
=======================================

For virtual deploys all the targets are VMs on the Jumpserver. The deploy script will:

   - Create a Salt Master VM on the Jumpserver which will drive the installation
   - Create the bridges for networking with virsh (only if a real bridge does not already exist for a given network)
   - Install OpenStack on the targets
      - Leverage Salt to install & configure OpenStack services

.. figure:: img/fuel_virtual.png
   :align: center
   :alt: Fuel@OPNFV Virtual POD Network Layout Examples

   Fuel@OPNFV Virtual POD Network Layout Examples

   +-----------------------+------------------------------------------------------------------------+
   | cfg01                 | Salt Master VM                                                         |
   +-----------------------+------------------------------------------------------------------------+
   | ctl01                 | Controller VM                                                          |
   +-----------------------+------------------------------------------------------------------------+
   | cmp001/cmp002         | Compute VMs                                                            |
   +-----------------------+------------------------------------------------------------------------+
   | gtw01                 | Gateway VM with neutron services (dhcp agent, L3 agent, metadata, etc) |
   +-----------------------+------------------------------------------------------------------------+
   | odl01                 | VM on which ODL runs (for scenarios deployed with ODL)                 |
   +-----------------------+------------------------------------------------------------------------+


In this figure there are examples of two virtual deploys:
   - Jumphost 1 has only virsh bridges, created by the deploy script
   - Jumphost 2 has a mix of Linux and virsh bridges; When Linux bridge exists for a specified network,
     the deploy script will skip creating a virsh bridge for it

.. NOTE::

    A virtual network ``mcpcontrol`` is always created for initial connection of the VMs on Jumphost.


Automatic Installation of a Baremetal POD
=========================================

The baremetal installation process can be done by editing the information about
hardware and environment in the reclass files, or by using the files Pod Descriptor
File (PDF) and Installer Descriptor File (IDF) as described in the OPNFV Pharos project.
These files contain all the information about the hardware and network of the deployment
that will be fed to the reclass model during deployment.

The installation is done automatically with the deploy script, which will:

   - Create a Salt Master VM on the Jumpserver which will drive the installation
   - Create a MaaS Node VM on the Jumpserver which will provision the targets
   - Install OpenStack on the targets
      - Leverage MaaS to provision baremetal nodes with the operating system
      - Leverage Salt to configure the operating system on the baremetal nodes
      - Leverage Salt to install & configure OpenStack services

.. figure:: img/fuel_baremetal.png
   :align: center
   :alt: Fuel@OPNFV Baremetal POD Network Layout Example

   Fuel@OPNFV Baremetal POD Network Layout Example

   +-----------------------+---------------------------------------------------------+
   | cfg01                 | Salt Master VM                                          |
   +-----------------------+---------------------------------------------------------+
   | mas01                 | MaaS Node VM                                            |
   +-----------------------+---------------------------------------------------------+
   | kvm01..03             | Baremetals which hold the VMs with controller functions |
   +-----------------------+---------------------------------------------------------+
   | cmp001/cmp002         | Baremetal compute nodes                                 |
   +-----------------------+---------------------------------------------------------+
   | prx01/prx02           | Proxy VMs for Nginx                                     |
   +-----------------------+---------------------------------------------------------+
   | msg01..03             | RabbitMQ Service VMs                                    |
   +-----------------------+---------------------------------------------------------+
   | dbs01..03             | MySQL service VMs                                       |
   +-----------------------+---------------------------------------------------------+
   | mdb01..03             | Telemetry VMs                                           |
   +-----------------------+---------------------------------------------------------+
   | odl01                 | VM on which ODL runs (for scenarios deployed with ODL)  |
   +-----------------------+---------------------------------------------------------+
   | Tenant VM             | VM running in the cloud                                 |
   +-----------------------+---------------------------------------------------------+

In the baremetal deploy all bridges but "mcpcontrol" are Linux bridges. For the Jumpserver, it is
required to pre-configure at least the admin_br bridge for the PXE/Admin.
For the targets, the bridges are created by the deploy script.

.. NOTE::

    A virtual network ``mcpcontrol`` is always created for initial connection of the VMs on Jumphost.


Steps to Start the Automatic Deploy
===================================

These steps are common both for virtual and baremetal deploys.

#. Clone the Fuel code from gerrit

   For x86_64

   .. code-block:: bash

       $ git clone https://git.opnfv.org/fuel
       $ cd fuel

   For aarch64

   .. code-block:: bash

       $ git clone https://git.opnfv.org/armband
       $ cd armband

#. Checkout the Fraser release

   .. code-block:: bash

       $ git checkout opnfv-6.2.1

#. Start the deploy script

    Besides the basic options,  there are other recommended deploy arguments:

    - use ``-D`` option to enable the debug info
    - use ``-S`` option to point to a tmp dir where the disk images are saved. The images will be
      re-used between deploys
    - use ``|& tee`` to save the deploy log to a file

   .. code-block:: bash

       $ ci/deploy.sh -l <lab_name> \
                      -p <pod_name> \
                      -b <URI to configuration repo containing the PDF file> \
                      -s <scenario> \
                      -D \
                      -S <Storage directory for disk images> |& tee deploy.log

.. NOTE::

    The deployment uses the OPNFV Pharos project as input (PDF and IDF files)
    for hardware and network configuration of all current OPNFV PODs.
    When deploying a new POD, one can pass the ``-b`` flag to the deploy script to override
    the path for the labconfig directory structure containing the PDF and IDF (see below).

Examples
--------
#. Virtual deploy

   To start a virtual deployment, it is required to have the **virtual** keyword
   while specifying the pod name to the installer script.

   It will create the required bridges and networks, configure Salt Master and
   install OpenStack.

      .. code-block:: bash

          $ ci/deploy.sh -l ericsson \
                         -p virtual3 \
                         -s os-nosdn-nofeature-noha \
                         -D \
                         -S /home/jenkins/tmpdir |& tee deploy.log

   Once the deployment is complete, the OpenStack Dashboard, Horizon, is
   available at ``http://<controller VIP>:8078``
   The administrator credentials are **admin** / **opnfv_secret**.

   A simple (and generic) sample PDF/IDF set of configuration files may
   be used for virtual deployments by setting lab/POD name to ``local-virtual1``.
   This sample configuration is x86_64 specific and hardcodes certain parameters,
   like public network address space, so a dedicated PDF/IDF is highly recommended.

      .. code-block:: bash

          $ ci/deploy.sh -l local \
                         -p virtual1 \
                         -s os-nosdn-nofeature-noha \
                         -D \
                         -S /home/jenkins/tmpdir |& tee deploy.log

#. Baremetal deploy

   A x86 deploy on pod2 from Linux Foundation lab

      .. code-block:: bash

          $ ci/deploy.sh -l lf \
                         -p pod2 \
                         -s os-nosdn-nofeature-ha \
                         -D \
                         -S /home/jenkins/tmpdir |& tee deploy.log

      .. figure:: img/lf_pod2.png
         :align: center
         :alt: Fuel@OPNFV LF POD2 Network Layout

         Fuel@OPNFV LF POD2 Network Layout

   An aarch64 deploy on pod5 from Arm lab

      .. code-block:: bash

          $ ci/deploy.sh -l arm \
                         -p pod5 \
                         -s os-nosdn-nofeature-ha \
                         -D \
                         -S /home/jenkins/tmpdir |& tee deploy.log

      .. figure:: img/arm_pod5.png
         :align: center
         :alt: Fuel@OPNFV ARM POD5 Network Layout

         Fuel@OPNFV ARM POD5 Network Layout

   Once the deployment is complete, the SaltStack Deployment Documentation is
   available at ``http://<proxy public VIP>:8090``.

   When deploying a new POD, one can pass the ``-b`` flag to the deploy script to override
   the path for the labconfig directory structure containing the PDF and IDF.

   .. code-block:: bash

       $ ci/deploy.sh -b file://<absolute_path_to_labconfig> \
                      -l <lab_name> \
                      -p <pod_name> \
                      -s <scenario> \
                      -D \
                      -S <tmp_folder> |& tee deploy.log

   - <absolute_path_to_labconfig> is the absolute path to a local directory, populated
     similar to Pharos, i.e. PDF/IDF reside in ``<absolute_path_to_labconfig>/labs/<lab_name>``
   - <lab_name> is the same as the directory in the path above
   - <pod_name> is the name used for the PDF (``<pod_name>.yaml``) and IDF (``idf-<pod_name>.yaml``) files



Pod and Installer Descriptor Files
==================================

Descriptor files provide the installer with an abstraction of the target pod
with all its hardware characteristics and required parameters. This information
is split into two different files:
Pod Descriptor File (PDF) and Installer Descriptor File (IDF).

The Pod Descriptor File is a hardware description of the pod
infrastructure. The information is modeled under a yaml structure.
A reference file with the expected yaml structure is available at
``mcp/config/labs/local/pod1.yaml``.

The hardware description is arranged into a main "jumphost" node and a "nodes"
set for all target boards. For each node the following characteristics
are defined:

- Node parameters including CPU features and total memory.
- A list of available disks.
- Remote management parameters.
- Network interfaces list including mac address, speed, advanced features and name.

.. NOTE::

    The fixed IPs are ignored by the MCP installer script and it will instead
    assign based on the network ranges defined in IDF.

The Installer Descriptor File extends the PDF with pod related parameters
required by the installer. This information may differ per each installer type
and it is not considered part of the pod infrastructure.
The IDF file must be named after the PDF with the prefix "idf-". A reference file with the expected
structure is available at ``mcp/config/labs/local/idf-pod1.yaml``.

The file follows a yaml structure and two sections "net_config" and "fuel" are expected.

The "net_config" section describes all the internal and provider networks
assigned to the pod. Each used network is expected to have a vlan tag, IP subnet and
attached interface on the boards. Untagged vlans shall be defined as "native".

The "fuel" section defines several sub-sections required by the Fuel installer:

- jumphost: List of bridge names for each network on the Jumpserver.
- network: List of device name and bus address info of all the target nodes.
  The order must be aligned with the order defined in PDF file. Fuel installer relies on the IDF model
  to setup all node NICs by defining the expected device name and bus address.
- maas: Defines the target nodes commission timeout and deploy timeout. (optional)
- reclass: Defines compute parameter tuning, including huge pages, cpu pinning
  and other DPDK settings. (optional)

The following parameters can be defined in the IDF files under "reclass". Those value will
overwrite the default configuration values in Fuel repository:

- nova_cpu_pinning: List of CPU cores nova will be pinned to. Currently disabled.
- compute_hugepages_size: Size of each persistent huge pages. Usual values are '2M' and '1G'.
- compute_hugepages_count: Total number of persistent huge pages.
- compute_hugepages_mount: Mount point to use for huge pages.
- compute_kernel_isolcpu: List of certain CPU cores that are isolated from Linux scheduler.
- compute_dpdk_driver: Kernel module to provide userspace I/O support.
- compute_ovs_pmd_cpu_mask: Hexadecimal mask of CPUs to run DPDK Poll-mode drivers.
- compute_ovs_dpdk_socket_mem: Set of amount huge pages in MB to be used by OVS-DPDK daemon
  taken for each NUMA node. Set size is equal to NUMA nodes count, elements are divided by comma.
- compute_ovs_dpdk_lcore_mask: Hexadecimal mask of DPDK lcore parameter used to run DPDK processes.
- compute_ovs_memory_channels: Number of memory channels to be used.
- dpdk0_driver: NIC driver to use for physical network interface.
- dpdk0_n_rxq: Number of RX queues.


The full description of the PDF and IDF file structure are available as yaml schemas.
The schemas are defined as a git submodule in Fuel repository. Input files provided
to the installer will be validated against the schemas.

- ``mcp/scripts/pharos/config/pdf/pod1.schema.yaml``
- ``mcp/scripts/pharos/config/pdf/idf-pod1.schema.yaml``

=============
Release Notes
=============

Please refer to the :ref:`Release Notes <fuel-release-notes-label>` article.

==========
References
==========

OPNFV

1) `OPNFV Home Page <https://www.opnfv.org>`_
2) `OPNFV documentation <https://docs.opnfv.org>`_
3) `Software downloads <https://www.opnfv.org/software/download>`_

OpenStack

4) `OpenStack Queens Release Artifacts <https://www.openstack.org/software/queens>`_
5) `OpenStack Documentation <https://docs.openstack.org>`_

OpenDaylight

6) `OpenDaylight Artifacts <https://www.opendaylight.org/software/downloads>`_

Fuel

7) `Mirantis Cloud Platform Documentation <https://docs.mirantis.com/mcp/latest>`_

Salt

8) `Saltstack Documentation <https://docs.saltstack.com/en/latest/topics>`_
9) `Saltstack Formulas <https://salt-formulas.readthedocs.io/en/latest/develop/overview-reclass.html>`_

Reclass

10) `Reclass model <https://reclass.pantsfullofunix.net>`_
