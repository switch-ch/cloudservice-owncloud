Service Architecture at CESNET
===============================

The following picture shows the current architecture diagram for
ownCloud service at CESNET.

.. image:: images/architecture/CesnetOcArchitecture1.0.png

Our ownCloud architecture is built around HA (High Availability) cluster,
which is managed by Pacemaker_. The cluster consists of 5 equivalent nodes.
Pacemaker assigns owncloud and postgres service to one of the front-end nodes.
All nodes are physical non-virtualized machines.

Specs
------

Hardware specifications for all front-ends are as follows:

  * CPU: 2 x Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz (24 CPUs)
  * RAM: 96 GB
  * NET: 8 Gbps



Application Server
------------------

Database Server
---------------

High Availability
-----------------

User Authentication
-------------------

Data Storage
------------

Monitoring
----------

.. links
.. _Pacemaker: http://clusterlabs.org/quickstart-redhat.html

