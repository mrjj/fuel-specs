..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Support for plugin links in Environment and Equipment dashboards
================================================================

https://blueprints.launchpad.net/fuel/+spec/external-dashboard-links-in-fuel-dashboard

Extend Dashboard tab of operational OpenStack environment with links to
dashboards of plugins installed to the environment.

Extend Plugins and Equipment Fuel UI pages with links to dashboards of plugins
installed on master node and not related to any particular environment.

-------------------
Problem description
-------------------

For now dashboard links of installed Fuel plugins are not available for user
from Fuel UI, that is not good UX. Fuel UI should cover the feature.


----------------
Proposed changes
----------------

Web UI
======

#. Dashboard tab of operational environment should display a set of installed
   plugin dashboard links sorted by `id` attribute, if any.

   Each entry should display the following data:

   * plugin title (that a link to plugin dashboard)
   * plugin description
   * link to 'http' version of plugin link url id TLS is enabled in
     environment (see url processing logic below)
   * button to hide the plugin from the dashboard

   .. image:: ../../images/8.0/external-dashboard-links-in-fuel-dashboard/
      plugin_blocks.png
      :scale: 75 %

   The environment-level entries come from Nailgun in
   `GET /api/clusters/:cluster_id/plugin_links` response.

   If url provided in plugin entry is relative, then Fuel UI code should
   process it in the following way:

   * check `public_ssl.horizon` environment setting (comes in
     `GET /api/clusters/:cluster_id/attributes response`) value:

     * if `public_ssl.horizon` is True, then a protocol for result url should
       be `https` and hostname is a `public_ssl.hostname` environment setting
       value
     * if `public_ssl.horizon` is False, then a protocol for result url should
       be `http` and hostname is environment virtual IP (stored in environment
       network configuration data)

   No processing required for an absolute plugin url.

   To hide a plugin link user should be able to click its 'Hide' button
   (`hidden` attribute of the plugin link should be updated to True in DB
   using an appopriate PUT request).

   Horizon link block should be restyled and shown on environment dashboard
   as well as a plugin link.

#. Plugins page in Fuel UI should be also updated to display links to
   dashboards for master node plugins if it is provided. The links should be
   taken from `GET /api/plugins/:plugin_id/links` response.

   If a link metadata contains `group` attribute, then the dashboard link
   should be also provided in Fuel UI in location, specified by the attribute:

   * in case of `equipment` group, a plugin link should be shown on
     Equipment page also.

   On Equipment page a plugin link block should contain the following data:

   * plugin title (that a link to plugin dashboard)
   * plugin description

   If no group specified for a plugin, then its link will be shown on Plugins
   page only.

   No additional processing required for master node plugin dashboard url.

   Master node plugin dashboard links can not be hidden in Fuel UI.


Nailgun
=======

New API `api/clusters/:id/plugin_links` endpoint should be created to
support environment plugin dashboard links management.

Appropriate API `api/plugins/:id/links` should be created to support
management of plugin dashboard links that refer to dashboards on master node.

The plugins should have a possibility to create/update/delete their entries
which will be shown in Fuel UI.


Data model
----------

The new table for environment plugin dashboard links should be created in
Nailgun DB, containing the following fields:

+----------+----------+-------------+----------+----------+------------+
| id       | title    | description | url      | hidden   | cluster_id |
+==========+==========+=============+==========+==========+============+
| id       | String   | String      | String   | Boolean  | id         |
| required | required | default:    | required | default: | required   |
|          |          | None        |          | False    |            |
+----------+----------+-------------+----------+----------+------------+

Also quite similar table for master node plugin dashboard entries should also
be created:

+----------+----------+-------------+----------+----------+-----------+
| id       | title    | description | url      | hidden   | plugin_id |
+==========+==========+=============+==========+==========+===========+
| id       | String   | String      | String   | Boolean  | id        |
| required | required | default:    | required | default: | required  |
|          |          | None        |          | False    |           |
+----------+----------+-------------+----------+----------+-----------+


REST API
--------

To support management of environment plugin links Nailgun API should be
extended with the following methods:

+--------+-----------------------------+---------------------+-------------+
| method | URL                         | action              | auth exempt |
+========+=============================+=====================+=============+
|  POST  | /api/clusters/:cluster_id/  | create a new plugin | true        |
|        | plugin_links                | link                |             |
+--------+-----------------------------+---------------------+-------------+
|  GET   | /api/clusters/:cluster_id/  | get list of plugin  | false       |
|        | plugin_links                | links               |             |
+--------+-----------------------------+---------------------+-------------+
|  PUT   | /api/clusters/:cluster_id/  | update a plugin     | false       |
|        | plugin_links/:entry_id      | link                |             |
+--------+-----------------------------+---------------------+-------------+
| DELETE | /api/clusters/:cluster_id/  | delete a plugin     | false       |
|        | plugin_links/:entry_id      | link                |             |
+--------+-----------------------------+---------------------+-------------+

The methods should return the following statuses in case of errors:

* 400 Bad Request - in case of invalid data (missing field, wrong format)
* 404 Not found - in case of missing entry
* 405 Not Allowed - for `PUT /api/clusters/:cluster_id/plugin_links`

GET method returns JSON of the following format:

.. code-block:: json

   [
     {
       title: 'Zabbix',
       description: 'Zabbix is software that monitors ...',
       url: 'https://172.5.6.24:80/zabbix_dashboard',
       hidden: false,
       id: Number(identificator)
     },
     {
       title: 'Murano',
       description: 'Murano dashboard link ...',
       url: '/openstack/murano_dashboard',
       hidden: false,
       id: Number(identificator)
     },
     ...
   ]

POST method accepts data of the following format:

.. code-block:: json

   {
     title: 'My plugin',
     description: 'My awesome plugin',
     url: '/my_plugin'
   }

and return data of the same format as GET.

PUT method accepts data of the following format:

.. code-block:: json

   {
     id: Number(identificator),
     title: 'New plugin title'
   }

and returns:

.. code-block:: json

   {
     title: 'New plugin title',
     description: 'My awesome plugin',
     url: '/my_plugin',
     hidden: false,
     id: Number(identificator)
   }

DELETE method accepts data of the following format:

.. code-block:: json

   {
     id: Number(identificator)
   }

To support management of master node plugin links, Nailgun API should be
extended with the following methods:

+--------+--------------------------------+----------------------+-------+
| method | URL                            | action               | auth  |
|        |                                |                      | exempt|
+========+================================+======================+=======+
|  POST  | /api/v1/plugins/:plugin_id/    | create a new plugin  | false |
|        | links                          | link                 |       |
+--------+--------------------------------+----------------------+-------+
|  GET   | /api/v1/plugins/:plugin_id/    | get a list of        | false |
|        | links                          | plugin link          |       |
+--------+--------------------------------+----------------------+-------+
|  PUT   | /api/v1/plugins/:plugin_id/    | update a plugin link | false |
|        | links/:link_id                 |                      |       |
+--------+--------------------------------+----------------------+-------+
| DELETE | /api/v1/plugins/:plugin_id/    | delete a plugin link | false |
|        | links/:link_id                 |                      |       |
+--------+--------------------------------+----------------------+-------+

The methods should return the following statuses in case of errors:

* 400 Bad Request - in case of invalid data (missing field, wrong format)
* 404 Not found - in case of missing entry
* 405 Not Allowed - for `PUT /api/plugins/:plugin_id/links`

GET method returns JSON of the following format:

.. code-block:: json

   [
     {
       title: 'Zabbix',
       description: 'Zabbix is software that monitors ...',
       url: 'https://172.5.6.24:80/zabbix_dashboard',
       hidden: false,
       id: Number(identificator)
     },
     {
       title: 'Murano',
       description: 'Murano dashboard link ...',
       url: '/openstack/murano_dashboard',
       hidden: false,
       id: Number(identificator)
     },
     ...
   ]

POST method accepts data of the following format:

.. code-block:: json

   {
     title: 'My plugin',
     description: 'My awesome plugin',
     url: '/my_plugin'
   }

and return data of the same format as GET.

PUT method accepts data of the following format:

.. code-block:: json

   {
     id: Number(identificator),
     title: 'New plugin title'
   }

and returns:

.. code-block:: json

   {
     title: 'New plugin title',
     description: 'My awesome plugin',
     url: '/my_plugin',
     hidden: false
     id: Number(identificator)
   }

DELETE method accepts data of the following format:

.. code-block:: json

   {
     id: Number(identificator)
   }


Orchestration
=============

None


RPC Protocol
------------

None


Fuel Client
===========

None


Plugins
=======

Plugin framework should be extended to provide an ability for the plugin to
create/update/delete its entry to be displayed in environment dashboard or
on Plugins/Equipment pages in Fuel UI.

To specify an extra Fuel UI location (Equipment page), master node plugin can
provide an additional group in it's `groups` attribute.

The following groups which are recognized by Fuel UI, are possible:

* `equipment` - plugin link will be also displayed on the equipment
  panel (Equipment page)


Fuel Library
============

None


------------
Alternatives
------------

None


--------------
Upgrade impact
--------------

According to existing data model impact, an appropriate migration should be
created.
Environments of old releases should support the feature too.


---------------
Security impact
---------------

None


--------------------
Notifications impact
--------------------

None


---------------
End user impact
---------------

None


------------------
Performance impact
------------------

None


-----------------
Deployment impact
-----------------

None


----------------
Developer impact
----------------

None


---------------------
Infrastructure impact
---------------------

None


--------------------
Documentation impact
--------------------

Both plugin development documentation and user guides should be updated
accordingly to the change.


--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  vkramskikh (vkramskikh@mirantis.com)

Other contributors:
  jkirnosova (jkirnosova@mirantis.com)
  vsharshov (vsharshov@mirantis.com)
  astepanchuk (astepanchuk@mirantis.com)
  bdudko (bdudko@mirantis.com)
  ikutukov (ikutukov@mirantis.com)

QA engineer:
  omartsyniuk (omartsyniuk@mirantis.com)

Mandatory design review:
  vkramskikh (vkramskikh@mirantis.com)
  akislitsky (akislitsky@mirantis.com)


Work Items
==========

#. Nailgun DB and API changes to support environment plugin links management
#. Nailgun DB and API changes to support master node plugin links management
#. Plugin framework changes to support environment plugin links management
#. Plugin framework changes to support master node plugin links management
#. Fuel UI changes to display plugin links in operational environment
   dashboard
#. Fuel UI changes to display plugin links on Plugins/Equipment pages


Dependencies
============

None


-----------
Testing, QA
-----------

* Nailgun tests for the new API, DB changes and migration
* Tests for plugins to check they provide a plugin link data properly
* Manual testing
* Functional UI auto-tests should cover the feature


Acceptance criteria
===================

* User can access dashboards of installed environment plugins from Dashboard
  tab of the operational environment in Fuel UI
* User can access dashboards of installed master node plugins from
  Plugins page in Fuel UI and other locations (Equipment page), specified in
  plugin data (`groups` attribute)


----------
References
----------

* #fuel-dev on freenode
