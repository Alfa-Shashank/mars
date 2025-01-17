.. _k8s:

Run on Kubernetes
=================

Mars can run in clusters managed by `Kubernetes <https://kubernetes.io>`_. You
can use ``mars.deploy.kubernetes`` to set up a Mars cluster.

Basic steps
-----------
Mars uses image repository ``marsproject/mars`` by default. Each released
version of Mars has its image since v0.3.0. For instance, the image for version
0.3.0 is ``marsproject/mars:v0.3.0``.  If you need to build an image from
source, you may run the command below:

.. code-block:: bash

    bin/kube-image-tool.sh build

A docker image with Mars tagged with the current version will be built.

Then you need to make sure if you have correct client configurations for
Kubernetes by running

.. code-block::

    kubectl get nodes

If it reports an error, please consult documentations for kubernetes or your
cluster maintainer for more information.

As Mars uses Python to operate on Kubernetes, you also need to install
Kubernetes client for Python locally. It can be installed with ``pip`` or
``conda``:

.. code-block:: bash

    # install with pip
    pip install kubernetes
    # install with conda
    conda install -c conda-forge python-kubernetes

After all these steps we can create a Mars cluster with one supervisor and one
worker with kubernetes and run some jobs on it:

.. code-block:: python

    from kubernetes import config
    from mars.deploy.kubernetes import new_cluster
    import mars.tensor as mt

    cluster = new_cluster(config.new_client_from_config())

    # new cluster will start a session and set it as default one
    # execute will then run in the local cluster
    a = mt.random.rand(10, 10)
    a.dot(a.T).execute()

    # after all jobs executed, you can turn off the cluster
    cluster.stop()

When you want to use this cluster elsewhere, you can obtain ``namespace`` and
``endpoint`` from the custer object and create another
``KubernetesClusterClient``:

.. code-block:: python

    # obtain information from current cluster
    namespace, endpoint = cluster.namespace, cluster.endpoint

    # create a new cluster client
    from kubernetes import config
    from mars.deploy.kubernetes import KubernetesClusterClient

    cluster = KubernetesClusterClient(
        config.new_client_from_config(), namespace, endpoint)

Customizing cluster
-------------------
``new_cluster`` function provides several keyword arguments for users to define
the cluster. You may use the argument ``image`` to specify the image for all
Mars pods, or use the argument ``timeout`` to specify timeout of cluster
creation.  Arguments for scaling up and out of the cluster are also available.

Arguments for supervisors:

+----------------------+-----------------------------------------------------------+
| Argument             | Description                                               |
+======================+===========================================================+
| supervisor_num       | Number of supervisors in the cluster, 1 by default        |
+----------------------+-----------------------------------------------------------+
| supervisor_cpu       | Number of CPUs for every supervisor                       |
+----------------------+-----------------------------------------------------------+
| supervisor_mem       | Memory size for supervisors in the cluster, in bytes      |
|                      | or size units like ``1g``                                 |
+----------------------+-----------------------------------------------------------+
| supervisor_extra_env | A mapping of environment variables to set in supervisors  |
+----------------------+-----------------------------------------------------------+

Arguments for workers:

+--------------------+----------------------------------------------------------------+
| Argument           | Description                                                    |
+====================+================================================================+
| worker_num         | Number of workers in the cluster, 1 by default                 |
+--------------------+----------------------------------------------------------------+
| worker_cpu         | Number of CPUs for every worker                                |
+--------------------+----------------------------------------------------------------+
| worker_mem         | Memory size for workers in the cluster, in bytes or size units |
|                    | like ``1g``                                                    |
+--------------------+----------------------------------------------------------------+
| worker_spill_paths | List of spill paths for worker pods on hosts                   |
+--------------------+----------------------------------------------------------------+
| worker_cache_mem   | Size or ratio of shared memory for every worker. Details about |
|                    | memory management of Mars workers can be found in :ref:`memory |
|                    | tuning <worker_memory_tuning>` section.                        |
+--------------------+----------------------------------------------------------------+
| min_worker_num     | Minimal number of ready workers for ``new_cluster`` to return, |
|                    | ``worker_num`` by default                                      |
+--------------------+----------------------------------------------------------------+
| worker_extra_env   | A dict of environment variables to set in workers.             |
+--------------------+----------------------------------------------------------------+

For instance, if you want to create a Mars cluster with 1 supervisor and 100
workers, each worker has 4 cores and 16GB memory, and stop waiting when 95
workers are ready, you can use the code below:

.. code-block:: python

    from kubernetes import config
    from mars.deploy.kubernetes import new_cluster

    api_client = config.new_client_from_config()
    cluster = new_cluster(api_client, supervisor_num=1, worker_num=100, worker_cpu=4,
                          worker_mem='16g', min_worker_num=95)

Implementation details
----------------------
When ``new_cluster`` is called, it will create an independent `namespace
<https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/>`_
for all objects including roles, role bindings, pods and services. When the
user destroys the service, the whole namespace will be destroyed.

Supervisors and workers are created with `deployments
<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>`_.
Services discover supervisors by directly accessing Kubernetes API via the
default `service account
<https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/>`_.
Pod addresses and their readiness are read by workers to decide whether to
start. Meanwhile the client read statuses of all pods and check if all
supervisors and at least ``min_worker_num`` workers are ready.

The readiness of Mars services are decided by `readiness probes
<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/>`_
whose result can be obtained via Pod statuses. For supervisors and workers,
when the service starts, an extra port will be opened in the service and the
pod can detect it.

As the default service account does not have privilege to read pods in
Kubernetes API, we create `roles
<https://kubernetes.io/docs/reference/access-authn-authz/rbac/>`_ with
capability to read and watch pods using RBAC API, and then bind them to default
service accounts within the namespace before creating replication controllers.
This enables Mars containers to detect the status of other containers.

Mars uses `Kubernetes services
<https://kubernetes.io/docs/concepts/services-networking/service/>`_ to expose
its service. Currently only ``NodePort`` mode is supported, and Mars looks for
the web endpoint of the supervisor as its endpoint. ``LoadBalancer`` mode is
not supported yet.
