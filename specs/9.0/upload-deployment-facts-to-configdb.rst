..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Upload serialized deployment facts to ConfigDB service API
==========================================================

https://blueprints.launchpad.net/fuel/+spec/upload-deployment-facts-to-configdb

There are multiple levels of hierarchy in Hiera used by deployment tasks on
nodes. Some of those data exist only in YaML files on a node and can't be
accessed by 3rd party components.

With configuration database service, we can store serialized deployment data
for later use. It requires though that we can upload the data from nodes to
the service as a part of deployment process.

We propose to develop a deployment task to
upload the facts to external HTTP-based API
and add it to a plugin that enables
3rd-party LCM application with Fuel [1]_.

--------------------
Problem description
--------------------

The store for serialized deployment information (e.g. ConfigDB API
extension in Nailgun [2]_) allows 3rd party applications to access
it. It also allows alternative deployment/lifecycle management
solutions to synchronize their configrations with Fuel installer.

However, it doesn't solve the problem of getting the information
into the service, since the extension itself is a more or less
passive store.

The solution is needed to perform actual upload of required information
into the configuration database. It also must keep the data up to date
by synchronizing them upon every change applied to the environment.

Synchronization required in the following cases:

#. Deployment settings changed in Nailgun via UI/CLI/API.
   In this case, Nailgun DB will have the latest changes, and Nailgun API
   will respond with properly updated serialized deployment data [2]_.
   This data can be imported into ConfigDB directly by requesting
   the Nailgun API and sending result to ConfigDB API. They are
   accessible by Hiera as ``astute`` data source [3]_.

#. Deployment data changed due to changes made to the node (e.g. hardware
   updated, versions of packages updated, etc) outside the Fuel context.
   These changes are reflected in the serialized data generated and stored
   on the node itself, in YaML files:

  * ``/etc/hiera/globals.yaml`` - global configurations calculated by
    deployment task ``globals``.

  * ``/etc/hiera/override/plugins.yaml`` - plugin-specific overrides
    of parameters defined in data sources on higher levels (i.e.
    ``astute`` and ``globals``).

  * ``/etc/hiera/override/configuration.yaml`` - specific overrides
    for OpenStack configuration parameters which are not exposed
    by Nailgun directly.

  * ``/etc/hiera/override/<hostname>.yaml`` - node-level configurations
    that override the basic parameters from other sources.

#. Deployment data changed in 3rd party deployment/lifecycle management
   application (e.g. in Puppet Master's top-level manifests or in External
   Nodes Classifier application for the Puppet Master). Here we need
   to import data from the 3rd party application. This case is out of
   scope of the current proposal.

In this specification, we will focus on the use cases #1 and #2.

----------------
Proposed changes
----------------

The current deployment process implemented in Fuel installer assumes
that all actual deployment data is available to Puppet agent on a target
node locally as a set of YaML files in ``/etc/hiera`` directory.

By the time Puppet agent starts to execute actual deployment tasks,
all the configuration settings must be up to date. It means that we
can import the actual set of the deployment configuration data from
those files as a part of deployment process.

The following changes are proposed in scope of this specification:

* Create a deployment task that uploads the serialized
  deployment data from files in ``/etc/hiera`` of a target node to
  the corresponding resources in ConfigDB API endpoint (e.g.
  ``/environment/<:id>/node/<:id>/resource/<:datasource>/values``).
  See Orchestration_ section for details.

* Integrate the a task into the deployment graph using plugins
  mechanisms of Fuel. The task must run in the end of the deployment
  process to make sure that all the tasks that impact
  the deployment settings files are already executed, including:

  * ``globals``

  * ``override_configuration``

  * any plugin task that updates/overrides basic deployment settings

* Implement a pass of auth information for ConfigDB API
  extension of Nailgun API to the deployment task in question
  in a secure way.

**Deployment task details**

The deployment task shall have the following ID:

::

    upload_data_to_configdb


The task shall depend on the following tasks:

::

    globals
    upload_configuration
    pre_deployment_end

The task shall run at the Fuel Master node. Auth credentials for the
ConfigDB API shall be made available for it upon LCM plugin installation
from the following file:

::

    /var/www/nailgun/plugins/<lcm-plugin-name>/environment_config.yaml

That file will be created by the plugin builder and will contain metadata
of the LCM application, including auth credentials in question.

The task shall perform the following operations using ConfigDB API:

* Verify if the environment ``env_id`` exists in the ConfigDB API.

  * If not, create an environment with ``POST`` request. It should
    contain a list of data sources to create for the environment. See details
    in ConfigDB API specification [2]_.

  * The list of data sources is fetched from ``astute.yaml`` file,
    from parameter ``data_sources`` of Puppet Master LCM plugin's metadata.

* Read data from files in ``/etc/hiera`` directory via ``mcollective``
  client into internal variable of the deployment task.

* Upload data to ConfigDB API's data sources based on the filenames from which
  the data was read.

* Read the *effective data* from ConfigDB API (i.e. values calculated by
  merging uploaded data with configured overrides at all levels or hierarchy)
  and write the data to files in ``/etc/hiera`` directory via ``mcollective``
  client.

The latter function ensures that the deployment process accounts for data
overriden in ConfigDB. This is required for 2nd day operations based on
the Fuel workflow, specifically for scale-out opertaions (adding a node
to the cluster) and replacement of failed nodes.

**Auth mechanism details**

ConfigDB API extension as a part of Nailgun API
uses Keystone to verify and authorize users.

Before installing an environment with the 3rd-party LCM plugin, user must add
a service account for the plugin in that environment with Keystone CLI. For
example:

::

    $ keystone user-create --name=lcm-plugin --tenant=admin

While configuring environment, the user enables the LCM plugin and configures
the service account credentials for the plugin via Fuel UI or API.

Plugin uses these access creates credentials to configure deployment task
``upload_data_to_configdb`` and the custom Hiera backend.

**Example workflow**

The following example illustrates the workflow of
the solution:

* Assume that the User intends to use 3rd-party
  application to perform some tasks, for example,
  LCM operations, on an OpenStack environment deployed
  by Fuel.

    * User installs the Fuel Master node with the
      ConfigDB extension. The extension is installed
      as an RPM on top of the existing system.

    * User installs a plugin for LCM operations that
      should include components to upload deployment
      data to ConfigDB API (e.g. deployment task
      ``upload_data_to_configdb``) and to
      perform lookup for certain parameters in ConfigDB
      API (e.g. custom Hiera backend).

* User configures OpenStack environment using Fuel UI.
  Nailgun creates metadata for the environment
  and individual nodes.

    * The deployment data for the
      environment and nodes is accessible via Nailgun
      API by URIs ``/cluster/<:id>/orchestrator/deployment``
      and ``/cluster/<:id>/orchestrator/deployment/default``.

    * Deployment data for specific node are exposed
      via the same URIs with addition of parameter
      ``?node=<:node_id>`` to the URI path.

* User deploys the environment as usual via Fuel
  UI or CLI.

    * Deployment task ``upload_data_to_configdb``
      runs on every node in the environment and
      uploads serialized deployment data from
      YaML files in ``/etc/hiera/`` directory to
      ConfigDB API.

    * Another deployment task configures the node
      to work with 3rd party LCM tool. This might
      or might not include disable of the ordinary
      Fuel means of deployment.

* Afterwards the User makes changes to
  the environment configuration using 3rd-party
  LCM tool.

    * User changes or extends the deployment
      settings by assigning values to parameters via
      ConfigDB API, for example, changes ``keystone_url``
      parameter in ``globals`` data source.

    * ConfigDB saves the override data to an override for the
      data source ``globals``.

    * User triggers 3rd party application which reads the *effective data*
      (i.e. raw uploaded values with applied overrides) from ConfigDB API
      and applies changes to all affected nodes.

* In future, the User adds another node to the
  environment and deploys it using standard Fuel
  methods.

    * Deployment data for the new node provided by
      Nailgun's standard serializers.

    * When the deployment is initiated, the task ``upload_data_to_configdb``
      synchronizes ``astute.yaml`` file created at the node by Astute with
      data overrides created in ConfigDB API: it uploads the contents of
      the file to ConfigDB API and then downloads *effective data* for the
      ``asuste`` data source from there.

    * After the pre-deployment finishes, the prepared deployment
      data are synchronized to the ConfigDB API by task
      ``upload_data_to_configdb``. This ensures that override settings
      from all data sources in the ConfigDB API are applied to all files
      in ``/etc/hiera`` at the node.

    * Deployment is done by Fuel standard deployment tasks, but
      with settings adjusted with overrides from ConfigDB.

    * After successful deployment, the LCM plugin reconfigures the node to
      work with 3rd-party LCM tool.

Web UI
======

None.

Nailgun
=======

None.

Data model
----------

None.

REST API
--------

None.

Orchestration
=============

A new deployment task shall be added to ensure
that all changes to files in ``/etc/hiera`` directory
are synchronized with the ConfigDB.

The task shall send a series of requests to the URI of the
resource in ConfigDB based on the parameters
of the deployment:

::

  <:service_uri>/environments/<:env_id>/nodes/<:node_id>/resources/<:datasource>/values

* ``service_uri`` is a endpoint from Keystone Service Catalog,
  defaults to ``/api/v1/config``.

* ``env_id`` is an identifier of cluster the node belongs to.
  The ID of environment shall be fetched
  from deployment fact ``deployment_id``.

* ``node_id`` is an identifier of the node. It should be matched in
  Nailgun API by the node's ``FQDN`` scope recieved from Puppet Master.

* ``datasource`` is a name of the data source.

The task will:
#. upload data to ``/values`` of the data source's resource and

#. download *effective data* from ``/values?effective`` and write it to files.

See detailed description of the API in corresponding
specification. [2]_

RPC Protocol
------------

None.

Fuel Client
===========

None.

Fuel Library
============

None.

------------
Alternatives
------------

The alternative way to keep deployment data from nodes in
sync with ConfigDB is to upload data to API from deployment tasks.

While it is possible to adjust ``globals`` and ``openstack_config``
tasks to upload configuration data to external service, it is
generally impossible to do with all supported plugins.

A plugin can override default values in ``astute.yaml``
generated by the Nailgun-provided serialized data. However,
this overrides are configured by plugin tasks
on a per-node basis. Override information is not available
to Nailgun or even Astute directly. So, to ensure sync
of plugins' override data we need to modify each and every plugin,
which apparently is not an option.

Another way to keep data in sync is to upload it from some
bottom-level catch-all Astute post-deployment task. This
would allow to keep Nailgun/ConfigDB credentials limited to
the Master node and not expose them to target nodes
in the deployment.

On the other hand, there was a work done on Astute to
convert its tasks into standard deployment tasks in
``fuel-library``. Thus, we should net add new tasks
to Astute in this proposal.

--------------
Upgrade impact
--------------

None.

---------------
Security impact
---------------

Sensitive configuration data, such as passwords and access credentials,
shall be uploaded to the ConfigDB API using proposed functions.
It is recommended to use encrypted HTTP protocol to
transfer these data.

The approach to authentication of the plugin's application with Nailgun API
assumes that the user is responsible for configuring access credentials
for the plugin applications in Keystone. The user is also responsible for
configuring proper credentials for the plugin when installing an environment
with 3rd-party LCM application.

--------------------
Notifications impact
--------------------

None.

---------------
End user impact
---------------

None.

------------------
Performance impact
------------------

The deployment task proposed in this spec will take
some time to upload all data to the ConfigDB API.
Moreover, if many nodes trying to write to the same
API endpoint at the same time, it might significantly
affect the overall duration of deployment.

-----------------
Deployment impact
-----------------

None.

----------------
Developer impact
----------------

None.

---------------------
Infrastructure impact
---------------------

The deployment task is packaged as a part of 3rd-party LCM plugin.

--------------------
Documentation impact
--------------------

None.

--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  <gelbuhos> Oleg S. Gelbukh

Other contributors:
    <sryabin> Sergey Ryabin

Mandatory design review:
  <rustyrobot> Evgeniy Li
  <ikalnitsky> Igor Kalnitsky
  <vsharshov> Vladimir Sharshov
  <vkuklin> Vladimir Kuklin

Work Items
==========

* Develop deployment task as a part of Puppet Master LCM
  plugin code base [1]_.

* Develop unit tests for the deployment task in the
  plugin's code base.

* Develop automated integration tests for the plugin in
  ``openstack/fuel-qa`` repository.

Dependencies
============

#. ConfigDB API implementation as Nailgun extension [2]_

------------
Testing, QA
------------

* The feature shall be tested in conjunction with
  ConfigDB API feature [2]_

* Tests shall verify that contents of data sources
  are consistent with contents of files in ``/etc/hiera``
  at nodes after the deployment finishes.

Acceptance criteria
===================

* Deployment data from nodes uploaded to corresponding
  data sources in ConfigDB API upon successful
  deployment of the OpenStack environment.

----------
References
----------

.. [1] Puppet Master LCM plugin specification TBD
.. [2] Nailgun API extension for serialized deployment facts https://review.openstack.org/#/c/284109/
.. [3] Nailgun API for Deployment Information https://github.com/openstack/fuel-web/blob/master/nailgun/nailgun/api/v1/handlers/orchestrator.py#L190
