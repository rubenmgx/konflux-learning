In Konflux, **etcd serves as the critical, highly-available key-value store that functions as the Kubernetes cluster's backing store for all cluster data** [previous API server response]. It is fundamental for managing the state of all API objects, including `Applications`, `Components`, `PipelineRuns`, and other resources within Konflux. The API server, which is the front-end of the Kubernetes control plane, persists the state of these objects in the `etcd` database [previous API server response].

Here's how `etcd` is used and its associated context in Konflux:

*   **Core Role and Importance**:
    *   `etcd` is essential for the cluster's operation, storing configuration data, state data, and metadata for Kubernetes objects [previous API server response].
    *   The team actively works to "keep etcd alive" because the **input to the cluster can be higher than what it can process**, sometimes leading to `etcd` becoming full of useful data even with regular maintenance operations like "compact+defrag".

*   **Observed Issues and Impact**:
    *   A recurring and significant problem is the `etcdserver: mvcc: database space exceeded` error. This error indicates that `etcd` has reached its storage capacity.
    *   When this issue occurs, it acts as an "infra limitation" that can **stall all new runs** and effectively causes a "cluster outage symptom" that blocks pipeline execution [previous etcd-shield response].
    *   "Etcd issues" have appeared and reappeared on clusters like `rh01`, manifesting differently but consistently causing problems. This is a "known issue" related to the Kubernetes `etcd` database "exceeding the 8GB maximum size".
    *   The high load on `etcd` has directly impacted other services, such as the **Renovate bot schedule, which was slowed down** because the number of pipelines it was starting was "exacerbating issue" for `etcd`.

*   **Mitigation**:
    *   Immediate actions to address `etcd` capacity issues include **initiating "compaction and defrag"** operations. There have been instances where an "etcd outage seems to be resolved" or was "fixed" after interventions.
    *   The `etcd-shield` mechanism was specifically designed as a **protection mechanism to safeguard the `etcd` database from reaching its capacity** due to high load, acting as a temporary measure [previous etcd-shield response].


More Information:

### What is etcd?

**etcd** is a distributed, reliable **key-value store** designed to consistently and safely store the configuration data and state of a distributed system. In Kubernetes, it serves as the **"single source of truth,"** holding the desired state of every object, such as Pods, Deployments, and Services. The name "etcd" is a combination of **"etc"** (from the Unix directory for system-wide configuration files) and **"d"** for "distributed."

***

### Key Components

etcd is built on a few core concepts that enable its reliability and consistency:

* **Raft Consensus Protocol:** etcd uses the **Raft consensus algorithm** to ensure that all members of an etcd cluster agree on the same data. Raft handles leader election and log replication, making the cluster highly resilient to failures.
* **Key-Value Store:** At its core, etcd is a simple data store. It stores data in a hierarchical format, similar to a file system, using keys and values. For example, a Kubernetes Pod might be stored at a key like `/registry/pods/default/nginx-pod`.
* **Watch API:** This is a crucial feature that allows clients (like the Kubernetes API server's controllers) to **watch for changes** to specific keys or directories. When a change occurs, etcd sends an event to the watching client. This push-based model is what enables the real-time reconciliation loop in Kubernetes.
* **Leases:** Leases are used to manage resource expiration. You can attach a lease to a key, and if the lease expires or is revoked, the corresponding key is automatically deleted.

***

### How is etcd built, something to have in consideration?

etcd is built to be a simple, secure, and robust component. Key considerations for its architecture include:

* **High Availability:** For production environments, etcd is deployed as a **cluster of at least three members** to tolerate failures and maintain a quorum. An odd number of members is recommended to prevent split-brain scenarios.
* **Consistency over Availability (CAP Theorem):** etcd prioritizes strong consistency, meaning all members of the cluster will have the same data at all times. This is essential for a system like Kubernetes, where the state must be accurate.
* **Secure by Default:** All client-server and peer-to-peer communication in etcd should be secured with **TLS/SSL encryption** to prevent man-in-the-middle attacks. Authentication and authorization are also supported to control access.
* **Performance:** etcd is optimized for small, frequent updates. While it can store large values, its performance is best when it's used for configuration data rather than large binaries or files.

***

### How does etcd work?

etcd works by maintaining a consistent, replicated log of all changes.

1.  **Request from API Server:** When the Kubernetes API server receives a request to create a resource, it first validates and mutates the request.
2.  **Raft Consensus:** The API server then sends a write request to a leader in the etcd cluster. The leader replicates the change to its followers. Once a **quorum** (a majority) of the followers acknowledge the change, the leader commits it to the distributed log.
3.  **Data Persistence:** The committed change is then saved to disk on all participating etcd nodes.
4.  **Watch Notification:** At the same time, because other Kubernetes controllers (like the scheduler or kubelet) are "watching" etcd for changes, etcd sends a notification event to them. This triggers the controllers to take action to make the cluster's actual state match the newly recorded desired state.

***

### Which would be a normal use case for etcd?

etcd's primary and most important use case is being the **backend data store for Kubernetes**. Every object you create in Kubernetes is stored as a key-value pair in etcd. For example:

* **Storing a Deployment:** When you create a Deployment, its configuration (including the number of replicas, container image, etc.) is written to etcd.
* **Storing Pod Status:** The `kubelet` on each node reports the status of its Pods back to the API server, which then updates the Pod's status in etcd.
* **Service Discovery:** Services and Endpoints are also stored in etcd, allowing them to be discovered by other components.

***

### What should I have in consideration to monitor to ensure functionality?

Monitoring etcd is crucial because if it fails, the entire Kubernetes cluster effectively stops working. You should monitor:

* **Leader Health:** Monitor the number of etcd leaders and a leader's uptime to ensure the leader is stable. Frequent leader changes could indicate network issues or instability.
* **Network and Replication Performance:** Track metrics related to peer communication and replication latency. High latency can impact the entire cluster.
* **Read/Write Performance:** Monitor the latency of client requests to ensure that read and write operations are fast enough. High latency could be caused by slow disks or a high number of requests.
* **Disk I/O:** etcd's performance is heavily dependent on disk speed. Monitor disk write and sync duration to ensure it can keep up with the write load.
* **Resource Usage:** Monitor the CPU, memory, and disk usage of the etcd pods. If etcd runs out of resources, it can become unstable or lose its quorum.