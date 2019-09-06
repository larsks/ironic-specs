..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Adding Policy Targets for Ironic Nodes
======================================

Include the URL of your StoryBoard story (which should have an `rfe`` tag):

https://storyboard.openstack.org/#!/story/XXXXXXX

This spec describes an update to the Ironic node controller to expose Owner
node data for policy checks.


Problem description
===================

Ironic is not multi-tenant; anyone with API access to one node
has access to all nodes. Nodes have an ``owner`` field; however it is
purely informational and not tied into any form of access control. This
makes it impossible for an Ironic deployment consisting of nodes from
multiple hardware Owners to have each Owner only be able to administrate
their own nodes. This is especially problematic in large Ironic
deployments, where the task of monitoring all nodes is overwhelming.


Proposed change
===============

The Ironic API already has a way of controlling access to the REST API: a
policy engine [0]_ that is used throughout the REST API. However, when the node
controller uses the policy engine [1]_, it does so without passing in any
information about the node being checked. Other services such as Nova [2]_ and
Barbican [3]_ pass in additional information, and we propose doing the same. In
summary, we would like to:

* Assume that a node's ``owner`` field is set to the project_id of the owner's
  project.
* Update the node controller so that policy checks also pass in information
  about the node, including ``owner``
* Update Ironic's default generated policy file to include ``is_node_owner``
  rules, along with updated rules for ``node:*`` actions (Nova does the same
  [4]_).  For example:

   *  "is_node_owner": "project_id:%(node.owner)s"
   *  "baremetal:node:set_power_state": "rule:is_admin or rule:is_node_owner"

This update is enough to control access for most API functions - except for list
functions. For those, we propose updating the node list query so that non-admin
results are filtered against the ``owner`` field, in a manner analogous to other
OpenStack services.

Alternatives
------------

One alternative is to perform these checks at the database API level. However
that requires a new code pattern to be used on a large number of individual
functions.

Data model impact
-----------------

None

State Machine Impact
--------------------

None

REST API impact
---------------

None

Client (CLI) impact
-------------------

None

"ironic" CLI
~~~~~~~~~~~~

None

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

None

RPC API impact
--------------

None

Driver API impact
-----------------

None

Nova driver impact
------------------

None

Ramdisk impact
--------------

None

Security impact
---------------

This change exposes functionality to additional users. However this access
is constrained through Oslo policy, and can be adjusted as an adminstrator
desires.

Other end user impact
---------------------

None

Scalability impact
------------------

None

Performance Impact
------------------

None: although node data needs to be retrieved in order to pass that
information into a policy check, the controller functions already fetch
that information. [5]_

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignees:
* tzumainn - tzumainn@redhat.com
* larsks - lars@redhat.com

Work Items
----------

* Update node controller.
* Add documentation.
* Write tests

Dependencies
============

None

Testing
=======

Both unit tests and Tempest tests will be added.

Upgrades and Backwards Compatibility
====================================

N/A

Documentation Impact
====================

Additional documentation describing the possible applications of
using the ``node_owner`` policy roles will be added.

References
==========

.. [0] https://github.com/openstack/ironic/blob/master/ironic/common/policy.py
.. [1] https://github.com/openstack/ironic/blob/master/ironic/api/controllers/v1/node.py#L225
   Example of a current policy check. Note the use of ``cdict``; it is being passed in as both
   the ``target`` and the ``creds``.
.. [2] https://github.com/openstack/nova/blob/master/nova/api/openstack/compute/servers.py#L648-L652
   Example of Nova creating a ``target`` dictionary.
.. [3] https://github.com/openstack/barbican/blob/stable/rocky/barbican/api/controllers/__init__.py#L59-L72
   Example of Barbican creating a ``target`` dictionary.
.. [4] https://github.com/openstack/nova/blob/master/nova/policies/base.py#L27-L30
   Example of Nova defaulting a rule that uses information from a ``target`` dictionary.
.. [5] https://github.com/openstack/ironic/blob/master/ironic/api/controllers/v1/node.py#L227
