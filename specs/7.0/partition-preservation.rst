==================================================
Support for partition preservation during rollback
==================================================

https://blueprints.launchpad.net/fuel/+spec/rollback-partition-preservation

Support for partition preservation.

Problem description
===================

:First:

Currently Fuel performs full re-provisioning while performing rollback which
implies data deletion. For instance, it doesn't preserve virtual machine
image files, compute node log files, database files. That said it loses
valuable data during rollback.
So operator has to manually backup valuable data before rollback.

Proposed change
===============

The idea is to preserve certain partitions on the nodes during rollback while
fully reformat others. For instance, keep partition /var/lib/nova/instances
intact but create filesystem on / (root) from the scratch.

Proposed features
-----------------

* Allow keeping certain partitions with valuable data on the nodes during
  provision.

* Allow configuring a set of partitions to preserve using standard
  fuel CLI commands to download and modify disk.yaml.

Implementation model
--------------------

General overview
++++++++++++++++

On data preservation following use cases were identified:
1) keep Ceph data
2) keep Swift data
3) keep Nova instances cache
4) keep DB/logs/other custom partition types

Architectural design
++++++++++++++++++++

First off, here's how partitioning in Fuel 6.x works right now. There are two
components - VolumeManager (on nailgun's side) and Fuel agent (a process
inside bootstrap OS). Fuel agent is quite straightforward - it just executes
orders from VolumeManager. Fuel agent knows nothing about partitioning layout,
it doesn't contain any business logic. So it is VolumeManager which contains
that business logic.

The idea of Rollback Paritition Preservation is to keep data (vm images, logs,
db files, etc) during reprovisioning. So in terms of partitioning instead of
fully deleting partitions and creating a new partition table Fuel will
preserve data on some specific partitions (not all) while it still will
reformat others (like root partition).

Preserved partition must be located as separate mountpoints
Mountpont should be added to the openstack.yaml
in case when specific mountpoint doesn't exist

Workflow will be as following:

1) Add 'keep' or similar boolean flag to node disks settings
   in Nailgun API (i.e. disks.yaml):

   ``fuel node --node 1 --disk --download``
   ::

     cat node_1/disks.yaml

     - extra:
       - disk/by-id/scsi-SATA_QEMU_HARDDISK_QM00001
       - disk/by-id/ata-QEMU_HARDDISK_QM00001
       id: disk/by-path/pci-0000:00:01.1-scsi-0:0:0:0
       name: sda
       size: 101836
       volumes:
       - name: os
         size: 101836
         keep: true

   then upload modified disk.yaml

   ``fuel node --node 1 --disk --upload``
2) Mcollective agent (erase_node.rb module in Mcollective) should
   be modified to disable erasing partitions with 'keep: True'
   when node is deleted from environment.
3) Fuel agent should be modified to be able to preserve partition
4) Fuel agent should save partition table if there is a partition
   with 'keep' flag set into true. On the Nailgun side all changes
   and operations of partition table should be not be accessible.
5) Not preserved partition should be erased using new File System.

Delivery details
++++++++++++++++

In context of this blueprint VolumeManager will be modified in order
to handle new partition preservation parameter. Also, Fuel agent will
be potentially changed, but those changes will be minor. In addition to
VolumeManager and Fuel agent there will be small changes in nailgun in order
to pass keep flag (disk.yam) to VolumeManager.

Alternatives
------------

The alternative approach would be copying valuable data back and
forth before and after the rollback.
But that would drastically increase time needed for rollback.

REST API impact
---------------

As this blueprint is all about adding new configurable behaviour to
node reinstallation feature - it introduces parameter to disk.yaml
to REST API which performs default value as False.

Data model impact
-----------------

None

Upgrade impact
--------------

This feature allows to speed up upgrade process of Compute and
Storage nodes due to no need to move huge amount of data.
This helps to minimize impact of upgrade procedure on end user
workloads running in the cloud.

Security impact
---------------

Partition preservation must be disabled when node is being
decommissioned from environment. Otherwise, user data remaining on
preserved partitions can pose a security risk.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

This blueprint itself is about boosting speed of rollback
and migration operations

Plugin impact
-------------

None

Other deployer impact
---------------------

This feature can affect services which use files from preserved
partition. In this case puppet manifests should be modified
and conform this feature
All changes for this services should be described in
corresponding specs.


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

:Primary Assignee: Ivan Ponomarev

:QA: Veronika Krayneva

:Documentation: Peter Zhurba, Dmitry Klenov

:Reviewer: Vladimir Kuklin, Vladimir Kozhukalov

Work Items
----------

1. Pass preserve partitions parameter from disk.yaml to Nailgun
   (VolumeManager)

2. Adapt VolumeManager to take partition preservation flag and
   generate appropriate partition layout for Fuel agent

3. Adapt fuel-agent/manager taking into account preserved partitions


Dependencies
============

https://blueprints.launchpad.net/fuel/+spec/mos-rollback

Testing
=======


Reinstall single compute on HW with partition preservation:

1) Enable partition preservation in disks settings (disks.yaml) of the
   compute
2) Do reinstallation of the compute
3) Run OSTF tests set
4) Run Network check
5) Check data on partitions
6) Check availability preserved VM's

Reinstall single controller on HW with partition preservation

1) Enable partition preservation in disks settings (disks.yaml) of the
   controller
2) Do reinstallation of the controller
3) Run OSTF tests set
4) Run Network check
5) Check data on partitions
6) Check services data that have been preserved
   Services should normally works using preserved data


Documentation Impact
====================

Documentation should be improved with
information about Partition Preservation options.

References
==========

https://blueprints.launchpad.net/fuel/+spec/mos-rollback
https://blueprints.launchpad.net/fuel/+spec/rollback-partition-preservation
