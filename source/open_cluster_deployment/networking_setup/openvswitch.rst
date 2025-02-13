.. _openvswitch:

================================================================================
Open vSwitch Networks
================================================================================

This guide describes how to use the `Open vSwitch <http://openvswitch.org/>`__ network drivers. They provide network isolation using VLANs by tagging ports and basic network filtering using OpenFlow. Other traffic attributes that may be configured through Open vSwitch are not modified.

The VLAN ID will be the same for every interface in a given network, calculated automatically by OpenNebula. It may also be forced by specifying an ``VLAN_ID`` parameter in the :ref:`Virtual Network template <vnet_template>`.

.. warning:: This driver doesn't support Security Groups.

OpenNebula Configuration
================================================================================

The VLAN ID is calculated according to this configuration option :ref:`/etc/one/oned.conf <oned_conf>`:

.. code::

    #  VLAN_IDS: VLAN ID pool for the automatic VLAN_ID assigment. This pool
    #  is for 802.1Q networks (Open vSwitch and 802.1Q drivers). The driver
    #  will try first to allocate VLAN_IDS[START] + VNET_ID
    #     start: First VLAN_ID to use
    #     reserved: Comma separated list of VLAN_IDs or ranges. Two numbers
    #     separated by a colon indicate a range.

    VLAN_IDS = [
        START    = "2",
        RESERVED = "0, 1, 4095"
    ]

By modifying this section, you can reserve some VLANs so they aren't assigned to a Virtual Network. You can also define the first VLAN ID. When a new isolated network is created, OpenNebula will find a free VLAN ID from the VLAN pool. This pool is global and it's also shared with the :ref:`802.1Q Networks <hm-vlan>`.

The following configuration parameters can be adjusted in ``/var/lib/one/remotes/etc/vnm/OpenNebulaNetwork.conf``:

+--------------------------+----------------------------------------------------------------------------------+
|      Parameter           |                                   Description                                    |
+==========================+==================================================================================+
| ``:arp_cache_poisoning`` | Set to ``true`` to enable ARP Cache Poisoning Prevention Rules                   |
|                          | (effective only with IP/MAC spoofing filters enabled on Virtual Network).        |
+--------------------------+----------------------------------------------------------------------------------+
| ``:keep_empty_bridge``   | Set to ``true`` to preserve bridges with no virtual interfaces left.             |
+--------------------------+----------------------------------------------------------------------------------+
| ``:ovs_bridge_conf``     | *(Hash)* Options for Open vSwitch bridge creation                                |
+--------------------------+----------------------------------------------------------------------------------+

.. note:: Remember to run ``onehost sync -f`` to synchonize the changes to all the nodes.

.. _ovswitch_net:

Defining Open vSwitch Network
==============================

To create an Open vSwitch network, include the following information:

+-----------------------+------------------------------------------------------------------------------------+-------------------------------+
|       Attribute       |                                       Value                                        |   Mandatory                   |
+=======================+====================================================================================+===============================+
| ``VN_MAD``            | Set ``ovswitch``                                                                   | **YES**                       |
+-----------------------+------------------------------------------------------------------------------------+-------------------------------+
| ``PHYDEV``            | Name of the physical network device that will be attached to the bridge            | NO (unless using VLANs)       |
+-----------------------+------------------------------------------------------------------------------------+-------------------------------+
| ``BRIDGE``            | Name of the Open vSwitch bridge to use                                             | NO                            |
+-----------------------+------------------------------------------------------------------------------------+-------------------------------+
| ``VLAN_ID``           | The VLAN ID, will be generated if not defined and ``AUTOMATIC_VLAN_ID=YES``        | NO                            |
+-----------------------+------------------------------------------------------------------------------------+-------------------------------+
| ``AUTOMATIC_VLAN_ID`` | Ignored if ``VLAN_ID`` defined. Set to ``YES`` to automatically assign ``VLAN_ID`` | NO                            |
+-----------------------+------------------------------------------------------------------------------------+-------------------------------+
| ``MTU``               | The MTU for the Open vSwitch port                                                  | NO                            |
+-----------------------+------------------------------------------------------------------------------------+-------------------------------+

For example, you can define an *Open vSwitch Network* with the following template:

.. code::

    NAME    = "private4"
    VN_MAD  = "ovswitch"
    BRIDGE  = vbr1
    VLAN_ID = 50          # Optional
    ...

.. warning:: Currently, if IP Spoofing enabled, only one NIC per VM for the same Open vSwith network can be attached.

Multiple VLANs (VLAN trunking)
------------------------------

VLAN trunking is also supported by adding the following tag to the ``NIC`` element in the VM template or to the virtual network template:

-  ``VLAN_TAGGED_ID``: Specify a range of VLANs to tag, for example: ``1,10,30,32,100-200``.

.. _openvswitch_vxlan:

Using Open vSwitch on VXLAN Networks
====================================

This section describes how to use `Open vSwitch <http://openvswitch.org/>`__ on VXLAN networks. To use VXLAN you need to use a specialized version of the Open vSwitch driver that incorporates the features of the :ref:`VXLAN <vxlan>` driver. It's necessary to be familiar with these two drivers, their configuration options, benefits, and drawbacks.

The VXLAN overlay network is used as a base with the Open vSwitch (instead of regular Linux bridge) on top. Traffic on the lowest level is isolated by the VXLAN encapsulation protocol and Open vSwitch still allows second level isolation by 802.1Q VLAN tags **inside the encapsulated traffic**. The main isolation is always provided by VXLAN, not 802.1Q VLANs. If 802.1Q is required to isolate the VXLAN, the driver needs to be configured with user-created 802.1Q tagged physical interface.

This hierarchy is important to understand.

OpenNebula Configuration
------------------------

There is no configuration specific to this driver, except the options specified above and in the :ref:`VXLAN Networks <vxlan>` guide.

Defining an Open vSwitch - VXLAN Network
----------------------------------------

To create a network, include the following information:

+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| Attribute                   | Value                                                                   | Mandatory                                      |
+=============================+=========================================================================+================================================+
| ``VN_MAD``                  | Set ``ovswitch_vxlan``                                                  |  **YES**                                       |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``PHYDEV``                  | Name of the physical network device that will be attached to the bridge.|  **YES**                                       |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``BRIDGE``                  | Name of the Open vSwitch bridge to use                                  |  NO                                            |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``OUTER_VLAN_ID``           | The outer VXLAN network ID.                                             |  **YES** (unless ``AUTOMATIC_OUTER_VLAN_ID``)  |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``AUTOMATIC_OUTER_VLAN_ID`` | If ``OUTER_VLAN_ID`` has been defined, this attribute is ignored.       |  **YES** (unless ``OUTER_VLAN_ID``)            |
|                             | Set to ``YES`` if you want OpenNebula to generate an automatic ID.      |                                                |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``VLAN_ID``                 | The inner 802.1Q VLAN ID. If this attribute is not defined a VLAN ID    |  NO                                            |
|                             | will be generated if AUTOMATIC_VLAN_ID is set to YES.                   |                                                |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``AUTOMATIC_VLAN_ID``       | Ignored if ``VLAN_ID`` defined. Set to ``YES`` to automatically         |  NO                                            |
|                             | assign ``VLAN_ID``                                                      |                                                |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``MTU``                     | The MTU for the VXLAN interface and bridge                              |  NO                                            |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+

For example, you can define an *Open vSwitch - VXLAN Network* with the following template:

.. code::

    NAME          = "private5"
    VN_MAD        = "ovswitch_vxlan"
    PHYDEV        = eth0
    BRIDGE        = ovsvxbr0.10000
    OUTER_VLAN_ID = 10000               # VXLAN VNI
    VLAN_ID        = 50                 # Optional VLAN ID
    ...

In this example, the driver will check for the existence of bridge ``ovsvxbr0.10000``.  If it doesn't exist, it will be created. Also, the VXLAN interface ``eth0.10000`` will be created and attached to the Open vSwitch bridge ``ovsvxbr0.10000``. When a virtual machine is instantiated, its bridge ports will be tagged with 802.1Q VLAN ID ``50``.

.. _openvswitch_dpdk:

Open vSwitch with DPDK
================================================================================

This section describes how to use a DPDK datapath with the Open vSwitch drivers. When using the DPDK backend, the OpenNebula drivers will automatically configure the bridges and ports accordingly.

.. warning:: This section is only relevant for KVM guests.

Requirements & Limitations
--------------------------------------------------------------------------------

Please consider the following when using the DPDK datapath for Open vSwitch:

* An Open vSwitch version compiled with DPDK support is required.
* The VMs need to use the virtio interface for its NICs.
* Although not needed to make it work, you'd probably be interested in configuring NUMA pinning and hugepages in your Hosts. See :ref:`here <numa>`.

OpenNebula Configuration
--------------------------------------------------------------------------------

There are no special configuration on the OpenNebula server. Note that the sockets used by the vhost interface are created in the VM directory (``/var/lib/one/datastores/<ds_id>/<vm_id>``) and named after the switch port.

Using DPDK in your Virtual Networks
-----------------------------------

There are no additional changes, simply:

* Create your networks using the ``ovswitch`` driver, :ref:`see above <openvswitch>`.
* Change configuration of the ``BRIDGE_TYPE`` of the network to ``openvswitch_dpdk`` using either the CLI command ``onevnet update`` or Sunstone.
* Make sure that the NIC model is set to ``virtio``. This setting can be added as a default in ``/etc/one/vmm_exec/vmm_exec_kvm.conf``.

You can verify that the VMs are using the vhost interface by looking at their domain definition in the Host. You should see something like:

.. code:: bash

   <domain type='kvm' id='417'>
     <name>one-10</name>
     ...
     <devices>
       ...
       <interface type='vhostuser'>
         <mac address='02:00:c0:a8:7a:02'/>
         <source type='unix' path='/var/lib/one//datastores/0/10/one-10-0' mode='server'/>
         <target dev=''/>
         <model type='virtio'/>
         <alias name='net0'/>
         <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
       </interface>
     ...
   </domain>

And the associated port in the bridge using the qemu vhost interface:

.. code:: bash

    Bridge br0
        Port "one-10-0"
            Interface "one-10-0"
                type: dpdkvhostuserclient
                options: {vhost-server-path="/var/lib/one//datastores/0/10/one-10-0"}

.. _openvswitch_qinq:

Using Open vSwitch with Q-in-Q
================================================================================

Q-in-Q is an amendment to the IEEE 802.1Q specification that provides the capability for multiple VLAN tags to be inserted into a single Ethernet frame. Using Q-in-Q (aka C-VLAN, customer VLAN) tunneling allows to create Layer 2 Ethernet connection between customers cloud infrastructure and OpenNebula VMs, or use a single service VLAN to bundle different customer VLANs.

OpenNebula Configuration
------------------------

There is no configuration specific for this use case, just consider the general options specified above.

Defining a Q-in-Q Open vSwitch Network
----------------------------------------

To create a network you need to include the following information:

+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| Attribute                   | Value                                                                   | Mandatory                                      |
+=============================+=========================================================================+================================================+
| ``VN_MAD``                  | Set ``ovswitch``                                                        |  **YES**                                       |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``PHYDEV``                  | Name of the physical network device that will be attached to the bridge.|  **YES**                                       |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``BRIDGE``                  | Name of the Open vSwitch bridge to use                                  |  NO                                            |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``VLAN_ID``                 | The service 802.1Q VLAN ID. If not defined the VLAN ID tag              |  NO                                            |
|                             | will be generated if AUTOMATIC_VLAN_ID is set to YES.                   |                                                |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``AUTOMATIC_VLAN_ID``       | Ignored if ``VLAN_ID`` defined. Set to ``YES`` to automatically         |  NO                                            |
|                             | assign ``VLAN_ID``                                                      |                                                |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``CVLANS``                  | Customer VLAN IDs, as a comma separated list (ranges supported)         |  **YES**                                       |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``QINQ_TYPE``               | Tag Protocol Identifier (TPID) for the service VLAN tag. Use ``802.1ad``|  NO                                            |
|                             | for TPID 0x88a8 or ``802.1q`` for TPID 0x8100                           |                                                |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+
| ``MTU``                     | The MTU for the Open vSwitch port                                       |  NO                                            |
+-----------------------------+-------------------------------------------------------------------------+------------------------------------------------+

For example, you can define an *Open vSwitch - QinQ Network* with the following template:

.. code::

    NAME     = "qinq_net"
    VN_MAD   = "ovswitch"
    PHYDEV   = eth0
    VLAN_ID  = 50                 # Service VLAN ID
    CVLANS   = "101,103,110-113"  # Customer VLAN ID list

In this example, the driver will assign and create an Open vSwitch bridge and will attach the interface ``eth0`` it. When a virtual machine is instantiated, its bridge ports will be tagged with 802.1Q VLAN ID ``50`` and service VLAN IDs ``101,103,110,111,112,113``. The configuration of the port should be similar to the that of following example that shows the second (``NIC_ID=1``) interface port ``one-1-5`` for VM 5:

.. code::

    # ovs-vsctl list Port one-5-1

    _uuid               : 791b84a9-2705-4cf9-94b4-43b39b98fe62
    bond_active_slave   : []
    bond_downdelay      : 0
    bond_fake_iface     : false
    bond_mode           : []
    bond_updelay        : 0
    cvlans              : [101, 103, 110, 111, 112, 113]
    external_ids        : {}
    fake_bridge         : false
    interfaces          : [6da7ff07-51ec-40e9-97cd-c74a36e2c267]
    lacp                : []
    mac                 : []
    name                : one-5-1
    other_config        : {qinq-ethtype="802.1q"}
    protected           : false
    qos                 : []
    rstp_statistics     : {}
    rstp_status         : {}
    statistics          : {}
    status              : {}
    tag                 : 100
    trunks              : []
    vlan_mode           : dot1q-tunnel
