OCI : Open Container Initiative. 
### Init vs SideCar Container:
#### Init contianer :
The container which runs before our any application containers runs to do a certain tasks like migrations, setting configurations, or running dependency checks, etc.
Whenever kubelet sends the request to pod for the initialization that lets start the pod, it will first start the init container. 
`Starting of app container is totally dependent on init container. It shares the resources allocated to pod with app container. It works in sync with app conatiner as if app container restarts it will also get restart.`

#### Sidecar container: 
This runs all the with app container. It provides certain input & it takes certain output from the app container. It is also referred as helper container.

## Job:
A Kubernetes **Job** is ==an API object that creates one or more Pods to perform a finite task and ensures they successfully run to completion==. Unlike Deployments, which maintain continuous, long-running applications, Jobs are transient. They are ideal for batch operations, database migrations, backups, or script executions. 

## Control Plane Components
Node/VM which hosts administrative components.
#### kube-apiserver
The core component server that exposes the Kubernetes HTTP API. Port used by API-Server `8443`.

The Kubernetes API server validates and configures data for the api objects which include pods, services, replicationcontrollers, and others. The API Server services REST operations and provides the frontend to the cluster's shared state through which all other components interact.

`kube-apiserver is designed to scale horizontally—that is, it scales by deploying more instances. You can run several instances of kube-apiserver and balance traffic between those instances.`

#### etcd
Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data. 

Only API server is able to interact with etcd database and it will only have the authority to apply those changes in etcd database. If need to retrieve any information from etcd database then also api server is the one which does the interaction.

If your Kubernetes cluster uses etcd as its backing store, make sure you have a [back up](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster) plan for the data.

All Kubernetes objects are stored in etcd. Periodically backing up the etcd cluster data is important to recover Kubernetes clusters under disaster scenarios, such as losing all control plane nodes. The snapshot file contains all the Kubernetes state and critical information. In order to keep the sensitive Kubernetes data safe, encrypt the snapshot files.

Backing up an etcd cluster can be accomplished in two ways: etcd built-in snapshot and volume snapshot.

- etcd supports built-in snapshot.A snapshot may either be created from a live member with the `etcdctl snapshot save` command or by copying the `member/snap/db` file from an etcd [data directory](https://etcd.io/docs/current/op-guide/configuration/#--data-dir) that is not currently used by an etcd process.

- If etcd is running on a storage volume that supports backup, such as Amazon Elastic Block Store, back up etcd data by creating a snapshot of the storage volume.

##### Caution

If any API servers are running in your cluster, you should not attempt to restore instances of etcd. Instead, follow these steps to restore etcd:

- stop _all_ API server instances
- restore state in all etcd instances
- restart all API server instances

The Kubernetes project also recommends restarting Kubernetes components (`kube-scheduler`, `kube-controller-manager`, `kubelet`) to ensure that they don't rely on some stale data. In practice the restore takes a bit of time. During the restoration, critical components will lose leader lock and restart themselves.

#### kube-scheduler
Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.

Control plane component that watches for newly created [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) with no assigned [node](https://kubernetes.io/docs/concepts/architecture/nodes/), and selects a node for them to run on.

Factors taken into account for scheduling decisions include: individual and collective [resource](https://kubernetes.io/docs/reference/glossary/?all=true#term-infrastructure-resource) requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, and deadlines.

#### kube-controller-manager
Runs controllers to implement Kubernetes API behavior.

Control plane component that runs [controller](https://kubernetes.io/docs/concepts/architecture/controller/) processes.

Logically, each [controller](https://kubernetes.io/docs/concepts/architecture/controller/) is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.

There are many different types of controllers. Some examples of them are:

- Node controller: Responsible for noticing and responding when nodes go down.
- Job controller: Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion.
- EndpointSlice controller: Populates EndpointSlice objects (to provide a link between Services and Pods).
- ServiceAccount controller: Create default ServiceAccounts for new namespaces.

The above is not an exhaustive list.

#### cloud-controller-manager (optional)
Integrates with underlying cloud provider(s).

A Kubernetes [control plane](https://kubernetes.io/docs/reference/glossary/?all=true#term-control-plane) component that embeds cloud-specific control logic. The cloud controller manager lets you link your cluster into your cloud provider's API, and separates out the components that interact with that cloud platform from components that only interact with your cluster.

The cloud-controller-manager only runs controllers that are specific to your cloud provider. If you are running Kubernetes on your own premises, or in a learning environment inside your own PC, the cluster does not have a cloud controller manager.

As with the kube-controller-manager, the cloud-controller-manager combines several logically independent control loops into a single binary that you run as a single process. You can scale horizontally (run more than one copy) to improve performance or to help tolerate failures.

The following controllers can have cloud provider dependencies:

- Node controller: For checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding
- Route controller: For setting up routes in the underlying cloud infrastructure
- Service controller: For creating, updating and deleting cloud provider load balancers

## Node Components
Run on every node, maintaining running pods and providing the Kubernetes runtime environment:

#### kubelet
Ensures that Pods are running, including their containers.

An agent that runs on each [node](https://kubernetes.io/docs/concepts/architecture/nodes/) in the cluster. It makes sure that [containers](https://kubernetes.io/docs/concepts/containers/) are running in a [Pod](https://kubernetes.io/docs/concepts/workloads/pods/).

The [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) takes a set of PodSpecs ( PodSpec is a YAML or JSON object that describes a pod ) that are provided through various mechanisms and ensures that the containers described in those PodSpecs are running and healthy. The kubelet doesn't manage containers which were not created by Kubernetes.

Other than from an PodSpec from the apiserver, there are two ways that a container manifest can be provided to the Kubelet.

- File: Path passed as a flag on the command line. Files under this path will be monitored periodically for updates. The monitoring period is 20s by default and is configurable via a flag.

- HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint is checked every 20 seconds (also configurable with a flag).

#### kube-proxy
Maintains network rules on nodes to implement Services.

kube-proxy is a network proxy that runs on each [node](https://kubernetes.io/docs/concepts/architecture/nodes/) in your cluster, implementing part of the Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/) concept.

[kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) maintains network rules on nodes. These network rules allow network communication to your Pods from network sessions inside or outside of your cluster.

kube-proxy uses the operating system packet filtering layer if there is one and it's available. Otherwise, kube-proxy forwards the traffic itself.

If you use a [network plugin](https://kubernetes.io/docs/concepts/architecture/#network-plugins) that implements packet forwarding for Services by itself, and providing equivalent behavior to kube-proxy, then you do not need to run kube-proxy on the nodes in your cluster.

#### Container runtime
Software responsible for running containers. 

A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment.

Kubernetes supports container runtimes such as [containerd](https://containerd.io/docs/), [CRI-O](https://cri-o.io/#what-is-cri-o), and any other implementation of the [Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-node/container-runtime-interface.md).



