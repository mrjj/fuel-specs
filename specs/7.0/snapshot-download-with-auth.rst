..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Snapshot download with authentication
=====================================

https://blueprints.launchpad.net/fuel/+spec/snapshot-download-with-auth

Required authentication for downloading snapshots

Problem description
===================

It is possible to guess (by brute force) diagnostic snapshot name and as a
result get access to all logins and passwords.

Proposed change
===============

Diagnostic snapshot URL is currently handled by nginx, nailgun
is not involved here. So we need to reconfigure nginx so this URL will be
also handled by nailgun.

* On the nailgun side, we need to implement a new handler for diagnostic
  snapshots. This handler will check for authentication.

* Handler shouldn't actually serve snapshots but use XSendfile
  feature of nginx [#]_. So after authentication check it should respond
  with empty response with proper X-Accel-Redirect header.

* Nginx will do the rest and send the snapshot to the client.


Alternatives
------------

We could encrypt snapshot using asymmetric cryptography

Data model impact
-----------------

None


REST API impact
---------------

.. http:get:: /dump/(snapshot_name)

  Check for authentication. Returns empty response with X-Accel-Redirect header
  set to `snapshot_name` location on server.

  :reqheader X-Auth-Token: authentication token from keystone
  :statuscode 200: no error
  :statuscode 401: Unauthorized
  :statuscode 404: Not found - on non-existing snapshot


Upgrade impact
--------------

None

Security impact
---------------

The feature is intended to improve End User's security in matter of
unauthorized access to sensitive data.

Notifications impact
--------------------

None

Other end user impact
---------------------

User should be already authenticated when executing command in fuelclient:
::

  fuel snapshot

Performance Impact
------------------

None

Plugin impact
-------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Change will have impact on fuel-qa scripts. In order to make it work, we need
to change the way snapshots are downloaded [#]_.

Infrastructure impact
---------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sbrzeczkowski

Work Items
----------

* Create new API Handler for snapshots serving
* Add authentication before downloading snapshot in fuel-qa [2]_


Dependencies
============

None

Testing
=======

Integration tests are required for this change:

* try to download snapshot without authentication - should fail with 401
* try to download snapshot with authentication - should succeed with 200
* try to download non-existing snapshot - should fail with 404

Acceptance criteria
-------------------

The most important thing is to not let End User to download snapshot
without authentication.

Documentation Impact
====================

Snapshot download will not be possible in command-line HTTP clients
(like curl) without providing proper authentication token (from keystone)
in "X-Auth-Token" header. It might break down scripts which are doing it
this way, so it should be mentioned in the documentation.

References
==========
.. [#] http://wiki.nginx.org/XSendfile
.. [#] https://github.com/stackforge/fuel-qa/blob/stable/6.1/fuelweb_test/helpers/decorators.py#L43