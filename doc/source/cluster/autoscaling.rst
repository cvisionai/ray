.. _ref-autoscaling:

Cluster Autoscaling
===================

Basics
------

The Ray Cluster Launcher will automatically enable a load-based autoscaler. When cluster resource usage exceeds a configurable threshold (80% by default), new nodes will be launched up to the specified ``max_workers`` limit (specified in the cluster config). When nodes are idle for more than a timeout, they will be removed, down to the ``min_workers`` limit. The head node is never removed.

In more detail, the autoscaler implements the following control loop:

 1. It calculates the estimated utilization of the cluster based on the most-currently-assigned resource. For example, suppose a cluster has 100/200 CPUs assigned, but 20/25 GPUs assigned, then the utilization will be considered to be max(100/200, 15/25) = 60%.
 2. If the estimated utilization is greater than the target (80% by default), then the autoscaler will attempt to add nodes to the cluster.
 3. If a node is idle for a timeout (5 minutes by default), it is removed from the cluster.

The basic autoscaling config settings are as follows:

.. code::

    # An unique identifier for the head node and workers of this cluster.
    cluster_name: default

    # The minimum number of workers nodes to launch in addition to the head
    # node. This number should be >= 0.
    min_workers: 0

    # The autoscaler will scale up the cluster to this target fraction of resource
    # usage. For example, if a cluster of 10 nodes is 100% busy and
    # target_utilization is 0.8, it would resize the cluster to 13. This fraction
    # can be decreased to increase the aggressiveness of upscaling.
    # The max value allowed is 1.0, which is the most conservative setting.
    target_utilization_fraction: 0.8

    # If a node is idle for this many minutes, it will be removed. A node is
    # considered idle if there are no tasks or actors running on it.
    idle_timeout_minutes: 5

Multiple Node Type Autoscaling
------------------------------

Ray supports multiple node types in a single cluster. In this mode of operation, the scheduler will look at the queue of resource shape demands from the cluster (e.g., there might be 10 tasks queued each requesting ``{"GPU": 4, "CPU": 16}``), and tries to add the minimum set of nodes that can fulfill these resource demands. This enables precise, rapid scale up compared to looking only at resource utilization, as the autoscaler also has visiblity into the aggregate resource load.

The concept of a cluster node type encompasses both the physical instance type (e.g., AWS p3.8xl GPU nodes vs m4.16xl CPU nodes), as well as other attributes (e.g., IAM role, the machine image, etc). `Custom resources <configure.html>`__ can be specified for each node type so that Ray is aware of the demand for specific node types at the application level (e.g., a task may request to be placed on a machine with a specific role or machine image via custom resource).

Multi-node type autoscaling operates in conjunction with the basic autoscaler. You may want to configure the basic autoscaler accordingly to act conservatively (i.e., set ``target_utilization_fraction: 1.0``).

An example of configuring multiple node types is as follows `(full example) <https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/aws/example-multi-node-type.yaml>`__:

.. code::

    # Specify the allowed node types and the resources they provide.
    # The key is the name of the node type, which is just for debugging purposes.
    # The node config specifies the launch config and physical instance type.
    available_node_types:
        cpu_4_ondemand:
            node_config:
                InstanceType: m4.xlarge
            resources: {"CPU": 4}
            max_workers: 5
        cpu_16_spot:
            node_config:
                InstanceType: m4.4xlarge
                InstanceMarketOptions:
                    MarketType: spot
            resources: {"CPU": 16, "Custom1": 1, "is_spot": 1}
            max_workers: 10
        gpu_1_ondemand:
            node_config:
                InstanceType: p2.xlarge
            resources: {"CPU": 4, "GPU": 1, "Custom2": 2}
            max_workers: 4
            worker_setup_commands:
                - pip install tensorflow-gpu  # Example command.
        gpu_8_ondemand:
            node_config:
                InstanceType: p2.8xlarge
            resources: {"CPU": 32, "GPU": 8}
            max_workers: 2
            worker_setup_commands:
                - pip install tensorflow-gpu  # Example command.

    # Specify the node type of the head node (as configured above).
    head_node_type: cpu_4_ondemand

    # Specify the default type of the worker node (as configured above).
    worker_default_node_type: cpu_16_spot


The above config defines two CPU node types (``cpu_4_ondemand`` and ``cpu_16_spot``), and two GPU types (``gpu_1_ondemand`` and ``gpu_8_ondemand``). Each node type has a name (e.g., ``cpu_4_ondemand``), which has no semantic meaning and is only for debugging. Let's look at the inner fields of the ``gpu_1_ondemand`` node type:

The node config tells the underlying Cloud provider how to launch a node of this type. This node config is merged with the top level node config of the YAML and can override fields (i.e., to specify the p2.xlarge instance type here):

.. code::

    node_config:
        InstanceType: p2.xlarge

The resources field tells the autoscaler what kinds of resources this node provides. This can include custom resources as well (e.g., "Custom2"). This field enables the autoscaler to automatically select the right kind of nodes to launch given the resource demands of the application. The resources specified here will be automatically passed to the ``ray start`` command for the node via an environment variable. For more information, see also the `resource demand scheduler <https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/resource_demand_scheduler.py>`__:

.. code::

    resources: {"CPU": 4, "GPU": 1, "Custom2": 2}

The ``max_workers`` field constrains the number of nodes of this type that can be launched:

.. code::

    max_workers: 4

The ``worker_setup_commands`` field can be used to override the setup and initialization commands for a node type. Note that you can only override the setup for worker nodes. The head node's setup commands are always configured via the top level field in the cluster YAML:

.. code::

    worker_setup_commands:
        - pip install tensorflow-gpu  # Example command.
