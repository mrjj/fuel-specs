..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Change OpenStack configuration after deployment
===============================================

https://blueprints.launchpad.net/fuel/+spec/openstack-config-change

We need the possibility to change OpenStack services configuration after
deployment.

All necessary services should be reloaded/restarted automatically.

-------------------
Problem description
-------------------

Now in UI/CLI you can set only small amount of OpenStack configuration options.
Final config files includes parameters from many places:

   - hiera
   - auto-generated based on other parameters
   - auto-generated based on hardware (ex. cpu count)
   - default from osnailyfacter module
   - default from upstream modules

Currently, operators do not have Fuel provided automated tools to modify
OpenStack configuration options after a cloud is deployed.
For example, if an operator needs to configure things such as Host Aggregates
or change CPU overcommit settings on specific nodes he must manually apply
updates to the nodes. These configuration changes may be reverted by a
subsequent Fuel deployment action.
So, it is necessary to introduce a mechanism for user to propagate changes into
OpenStack configuration files in post-deployment stage that would:

   - allow to post changes into OpenStack configs
   - allow to apply changes to all nodes with given role (controller, compute,
     storage) or to a specific set of nodes
   - keep track of changes made to specific nodes (configurations applied) via
     hiera or another mechanism

----------------
Proposed changes
----------------

We will create new puppet resource named 'override_resources'.
This resource will handle overriding already existing resources and creating
not defined resources.

.. code-block:: puppet

 keystone_config {
   'DEFAULT/debug': {value => True}
 }
 override_resource {'keystone_config':
   data => {
      'DEFAULT/debug': {'value' => False},
      'DEFAULT/max_param_size': {'value' => 128}
   }
 }

In that case 'DEFAULT/debug' will be overridden to value False, and
'DEFAULT/max_param_size' will be created with value 128.

override_resource will be used in all 'top level' granulars ex.
osnailyfacter/modular/keystone/keystone.pp for 'keystone_config'.

.. code-block:: puppet

 $override_configuration = hiera_hash('configuration', {})
 override_resources {'keystone_config':
   data => $override_configuration['keystone_config']
 }

New parameter hash will be passed to override_resources from hiera.
We will create multiple hiera files to cover hierachical config overrides:

   - /etc/hiera/override/config/%{::fqdn}
   - /etc/hiera/override/config/role
   - /etc/hiera/override/config/cluster

This way config overrides from nailgun DB, can be uploaded to hiera directly
without any kind of sort or filter.

Hiera will do deeper merge of configuration overrides.

Hiera will deliver hierarchical structure of data. Config for node_id is
more important than config for role, which is more important than configuration
for cluster.

Later on this mechanism can be extended to cover any kind of config override
hierarchy.

Top level granulars used to override configuration will have new property
named 'refresh_on'.

.. code-block:: yaml

 ---
 - id: keystone
   type: puppet
   groups: [primary-controller, controller]
   required_for: [openstack-controller]
   requires: [openstack-haproxy, database, rabbitmq]
   refresh_on: [keystone_config]
   parameters:
     puppet_manifest:
        /etc/puppet/modules/osnailyfacter/modular/keystone/keystone.pp
     puppet_modules: /etc/puppet/modules
     timeout: 3600
   test_pre:
     cmd: ruby
        /etc/puppet/modules/osnailyfacter/modular/keystone/keystone_pre.rb
   test_post:
     cmd: ruby
        /etc/puppet/modules/osnailyfacter/modular/keystone/keystone_post.rb

This 'refresh_on' will be used by nailgun to run proper task when user change
OpenStack configuration after deployment.

We will extend Fuel API to be able to upload new configuration for OpenStack
services to hiera for given node (execute 'upload_config_overrides').
We will extend Fuel API to be able to execute proper granular task for given
OpenStack configuration parameters.

Operator should be able to upload new configuration only for given node,
set of nodes or with given role (ex. all computes).

We will introduce new granular task 'upload_config_overrides' which will be
executed in pre-deployment.
This task will upload hiera override/config.
This way we can override configuration before cluster is deployed.

Web UI
======

None. Configuration manipulation will be available only to advanced users via
CLI.

Nailgun
=======

Data model
----------

We need to store in DB information about configuration manipulation.
Each configuration change for given entity (--upload) should be stored in
separate row.

New table openstack_configs should be created. It will have structure
that is described next:

   - id - Integer
   - is_active - Boolean; we will use this field to find active configurations
   - config_type - Enum; type of config, possible values are:
      - node
      - role
      - cluster
   - cluster_id - Integer, reference to cluster
   - node_id - Integer, reference to node, if config_type=node
   - node_role - String, node's role value if config_type=role
   - created_at - DateTime, timestamp when record was created
   - config - JSON, openstack configuration dictionary

is_active field will be automatically managed by nailgun.
When user upload new data, old data for given entity (ex. node_role=compute)
will be mark as inactive (is_active=False) and new data will be marked as
active (is_active=True).

Configuration manipulation YAML format is described below.

Example:

.. code-block:: yaml

 ---
 configuration:
   nova_config:
     DEFAULT/debug:
       value: True
     DEFAULT/amqp_durable_queues:
       value: False
   keystone_config:
     DEFAULT/default_publisher_id:
       ensure: absent
     DEFAULT/crypt_strength:
       value: 6000

REST API
--------

API should allow to get/set information about configuration manipulation for
given node, set of nodes.

When operator choose execute API should execute automatically all necessary
granular tasks.

API should do validation for each call:

   - Check if uploaded data has YAML format.
   - Check if chosen nodes are already deployed (in case when user manually
     run 'execute')
   - Check if all passed configuration (puppet resources names) are supported.

Add POST ``/openstack-config/`` to upload config.

Add GET ``/openstack-config/[?<filter>=<value>[&...]]`` to list configurations.
The following filters can be applied to this request:

  - ``active=<bool>`` - list only active configurations.
  - ``nodes=<node_id>[,...]`` - list configurations only
    for the specified nodes.
  - ``cluster=<cluster_id>`` - list configurations only
    for the specified cluster.
  - ``role=<role>`` - list configurations only for the specified role.

Add GET ``/openstack-config/<ID>`` to download config.

Add DELETE ``/openstack-config/<ID>`` to delete config. It works
as a `soft-delete`, which doesn't really delete data, but
marks it as deleted (is_active=False).

Add PUT ``/openstack-config/execute`` to apply configuration.
Filters described above can be also aplied to this request.

Orchestration
=============

RPC Protocol
------------

None

Fuel Client
===========

Flow of configuration option manipulation:

#. upload YAML:

   fuel openstack-config --env 1 --node 1,2,3 --upload --file file.yaml

   fuel openstack-config --env 1 --role compute --upload --file file.yaml

   fuel openstack-config --env 1 --upload --file file.yaml

Upload action, will mark configuration as 'is_active' = True.
Always latest config is treated as active.
node and role parameters will be mutually exclusive.

#. download YAML:

   fuel openstack-config --id 1 --download

As we will store historical data in DB, we need possibility to download
any record from the past. For that we will use 'id' parameter.
ID is id of DB record.

#. list historical data

   fuel openstack-config --env 1 --node 1 --list

   fuel openstack-config --env 1 --role compute --list

   fuel openstack-config --env 1 --list

#. execute YAML

   fuel openstack-config --env 1 --node 1,2,3 --execute

   fuel openstack-config --env 1 --role compute --execute

   fuel openstack-config --env 1 --execute

Plugins
=======

It is possible that after plugin deployment, operator will override parameter
used by plugin.
But we should remember that this feature is designed only for advanced users.
Moreover plugin developer also can set 'refresh_on' in plugin tasks.

Fuel Library
============

We need to prepare new puppet resource responsible for overriding puppet
resources.
We need to modify all 'top level' granulars to override configuration for
each OpenStack service.
We need to make sure these granulars are idempotent or as close to idempotent
as possible so that they can be rerun without issue.

------------
Alternatives
------------

Instead of using new puppet resource (override_resources), we can start passing
hash from hiera to all OpenStack services.
This way if operator want to change options, he should upload (via API), new
configuration which will be uploaded to hiera with highest priority.
After that nailgun will simply reexecute proper granular tasks which will
change conf files.

   Cons:
      - Review/rewrite multiple puppet manifests to use hash.

   Pros:
      - No need to find 'top level' granulars.
      - No additional puppet resource.

--------------
Upgrade impact
--------------

None

---------------
Security impact
---------------

New API should have standard Fuel API authentication enabled.
It is possible that on some nodes operator will have different (unsafe)
configuration options set.

--------------------
Notifications impact
--------------------

None

---------------
End user impact
---------------

In some cases configuration manipulation can lead to service disruption.

This feature is designed on for advanced users, because there is possibility
to destroy running cluster.

------------------
Performance impact
------------------

In most cases none.

Different set of configuration on different nodes could be followed with hard
to debug performance problems.

-----------------
Deployment impact
-----------------

None

----------------
Developer impact
----------------

None

--------------------------------
Infrastructure impact
--------------------------------

None

--------------------
Documentation impact
--------------------

We need to prepare documentation which will describe this feature.

--------------------
Expected OSCI impact
--------------------

None

--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  Bartosz Kupidura (zynzel)

Other contributors:
  Oleksandr Saprykin (cutwater)
  Sergiy Slipushenko (sslypushenko)
  Maciej Relewicz (rlu)
  Mikhail Polenchuk (mpolenchuk)

QA engineers
  Ksenia Demina <kdemina@mirantis.com>

Work Items
==========

 * Extend API to allow to store and execute configuration manipulation YAML
 * Write override_resources puppet resource
 * Modify all 'top level' granulars
 * Write new granular task 'upload_config_overrides'

Dependencies
============

Some OpenStack services are configured not by dedicated puppet resource, but
with concat/file_line/exec, we will not be able to override configuration
created this way.

Some OpenStack services (Neutron) use multiple puppet resources to set
configuration in single file. We should work with neutron upstream to handle
this.

Some puppet manifests are not indepotent.
In case we will add 'override_resources' to not indepotent manifest we need to
ensure that puppet will not break cluster.

-----------
Testing, QA
-----------

 * Extend TestRail with Manual CLI cases for each of the configuration option:
      - upload YAML
      - download YAML
      - execute YAML
      - upload and execute YAML
 * Extend TestRail with Manual CLI cases for the next configuration options:
      - CPU overcommit ratio
      - Reconfigure Keystone to use LDAP backend instead of default SQL
      - Change ephemeral disk storage setting
      - Change VLAN range used by ML2
      - Enable/disable Nova quotas
 * Lead manual CLI testing for the new test cases
 * Create System tests for the new test cases

Acceptance criteria
===================

 * User is provided with interface (CLI + API calls) to modify OpenStack
   options after cloud is deployed
 * User is provided with interface (CLI + API calls) to modify OpenStack
   options before cloud is deployed
 * New test cases are executed succesfully
 * The testing report is provided

----------
References
----------
