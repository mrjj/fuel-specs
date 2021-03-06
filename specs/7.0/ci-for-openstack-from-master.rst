..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
CI for OpenStack from master
==========================================

https://blueprints.launchpad.net/fuel/+spec/ci-for-openstack-from-master

This specification describes CI for OpenStack from master - from building
packages to releasing tested community ISO that will deploy latest
OpenStack. Creating such process will benefit OpenStack developers as well as
Fuel developers.


Problem description
===================

Every OpenStack developer needs some tool to deploy latest code plus his
own work to perform functional testing. Fuel is the most sophisticated
OpenStack deployment tool and a great candidate for such work. But it is always
one step back regarding OpenStack releases it supports.


Proposed change
===============

Create CI analogous to our current CI for regular releases but targeted at
master branch of OpenStack.


Alternatives
------------

None.


Data model impact
-----------------

None.


REST API impact
---------------

None.


Upgrade impact
--------------

None.


Security impact
---------------

None.


Notifications impact
--------------------

None.


Other end user impact
---------------------

None.


Performance Impact
------------------

None.


Plugin impact
-------------

None.


Other deployer impact
---------------------

Community users get Fuel ISOs with master OpenStack.


Developer impact
----------------

Fuel developers need to monitor test results and continuously update
fuel-library to support master OpenStack. For every OpenStack master
compatibility change in the Fuel code, a reasonable effort should be
dedicated to making that change backwards compatible with the
currently supported stable OpenStack version.

A new branch called "future" will be created in fuel-library
repository. It is acceptable to put an OpenStack master compatibility
change in this branch, iff the only way to ensure backwards
compatibility is to introduce a conditional statement with a check for
specific OpenStack release. Every time a change is merged to a
"future" branch, the core reviewer approving the change is responsible
for rebasing the whole "future" branch onto the current head of master
branch of the corresponding Fuel git repository.


Infrastructure impact
---------------------

Creating CI for master OpenStack puts additional load on Jenkins masters and
slaves (`OSCI Jenkins`_, `Fuel Jenkins`_). The amount of load is comparable to
existing CI branches for releases.

New jobs will be created, similar to jobs for numbered releases. Existing jobs
will be not affected. Also it's necessary to create one additional type of
Jenkins jobs. Usually package is built when developer uploads and then submits
a change request. In case with master branch OpenStack code is synchronized
with upstream daily via git (without CRs). Thus we need to perform daily builds
of OpenStack packages. Jobs that perform this task are called autobuild-master
jobs.

A new kind of release appears: community ISO with master OpenStack.


Implementation
==============

Assignee(s)
-----------

* Primary assignee, package build: `Alexander Tsamutali`_
* ISO build: `Alexandra Fedorova`_
* ISO testing with fuel-qa_: `Timur Nurlygayanov`_
* ISO testing with Tempest and Rally: `Artur Kaszuba`_


Work Items
----------

* Create branch "future" in fuel-library repository.
* Create OSCI Jenkins jobs to build master branch of system packages,
  dependencies and OpenStack.

  + master.mos.build-deb-request, master.mos.build-rpm-request
      Build OpenStack packages after new patch set was uploaded.
  + master.mos.build-deb-deps-request,
      master.mos.build-rpm-deps-request Build OpenStack dependencies
      and system packages after new patch set was uploaded.
  + master.mos.build-deb, master.mos.build-rpm
      Build OpenStack packages after merge.
  + master.mos.build-deb-deps, master.mos.build-rpm-deps
      Build OpenStack dependencies and system packages after merge.
  + master.mos.autobuild, master.mos.autobuild-deb, master.mos.autobuild-rpm
      Build OpenStack packages every day.

* Create OSCI Jenkins jobs to copy packages to mirrors.

  + master.mos.publisher
      Publish built package in repository.

* Create OSCI Jenkins jobs to test packages.

  + master.mos.install-deb, master.mos.install-rpm
      Simple install test for packages built from patch set.

* Create Fuel Jenkins jobs to perform per-commit tests for fuel-library.

  + master.fuel-library.pkgs
      Deployment tests.

* Create Fuel Jenkins jobs to build ISOs.
* Create Fuel Jenkins jobs to test ISOs.
* Release tested ISOs via fuel-infra.org_.


Dependencies
============

Related to task of supporting master OpenStack in Fuel.


Testing
=======

Packages built with these jobs will be tested for installation
only. ISOs will be tested with most generic fuel-qa tests, Tempest and
Rally. Status of all jobs, including latest successful build of ISO,
will be available on Fuel Jenkins.


Documentation Impact
====================

Documentation about CI process, master ISOs should be
written. Announcement will be sent to OpenStack community mailing
lists.


References
==========

None.


.. _`OSCI Jenkins`: http://osci-jenkins.srt.mirantis.net
.. _`Fuel Jenkins`: http://ci.fuel-infra.org
.. _`Alexander Tsamutali`: https://launchpad.net/~astsmtl
.. _`Alexandra Fedorova`: https://launchpad.net/~afedorova
.. _`Timur Nurlygayanov`: https://launchpad.net/~tnurlygayanov
.. _`Artur Kaszuba`: https://launchpad.net/~akaszuba
.. _fuel-infra.org: http://fuel-infra.org
.. _fuel-qa: http://git.openstack.org/cgit/stackforge/fuel-qa
