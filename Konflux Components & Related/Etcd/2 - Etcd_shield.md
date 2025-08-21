Etcd-shield is a **critical protection mechanism** within Konflux clusters, specifically designed to **safeguard the `etcd` database from reaching its capacity due to high load**. It acts as a **temporary measure** until a more robust queuing feature for Tekton pipelines can be fully implemented.

Here's a breakdown of its context and usage in relation to core services:

*   **Purpose and Mechanism**:
    *   When the `etcd` database approaches critical levels of space utilization, **etcd-shield automatically activates to block new `PipelineRun` admissions**. This directly protects the core `etcd` service, which is vital for the cluster's operation and for Tekton's `PipelineRuns`.
    *   This prevents the cluster from being overloaded, which could lead to `etcdserver: mvcc: database space exceeded` errors, effectively causing a "cluster outage symptom" that blocks pipeline execution.
    *   Users attempting to initiate pipeline runs during an active etcd-shield period will receive messages indicating that "PipelineRun admission currently not allowed".

*   **Impact on Users and Operations**:
    *   A significant limitation is that there is **no automated re-run mechanism** for blocked pipelines; users must manually re-trigger their pipeline runs once the cluster load subsides and `etcd` returns to a healthy state. This manual intervention is recognized as a "not ideal" temporary solution.
    *   The activation of etcd-shield has been observed to **block critical processes**, such as QE testing and releases, particularly in staging environments.

More Information:

### What is etcd?

**etcd** is a distributed, reliable **key-value store** designed to consistently and safely store the configuration data and state of a distributed system. In Kubernetes, it functions as the cluster's **"single source of truth,"** holding the desired state of every object, such as Pods, Deployments, and Services. The name "etcd" is an abbreviation of **"etc"** (from the Unix directory where system-wide configuration files are stored) and **"d"** for "distributed."

***

### Key Components

etcd is built on a few core concepts that enable its reliability and consistency:

* **Raft Consensus Protocol:** etcd uses the **Raft consensus algorithm** to ensure that all members of an etcd cluster agree on the same data. Raft handles leader election and log replication, making the cluster highly resilient to failures.
* **Key-Value Store:** At its core, etcd is a simple data store. It stores data in a hierarchical format, similar to a file system, using keys and values. For example, a Kubernetes Pod might be stored at a key like `/registry/pods/default/nginx-pod`.
* **Watch API:** This is a crucial feature that allows clients (like the Kubernetes API server's controllers) to **watch for changes** to specific keys or directories. When a change occurs, etcd sends an event to the watching client. This push-based model is what enables the real-time reconciliation loop in Kubernetes.
* **Leases:** Leases are used to manage resource expiration. You can attach a lease to a key, and if the lease expires or is revoked, the corresponding key is automatically deleted. This is used by the API server to handle a Pod's TTL (Time-to-Live).

***

### How is etcd built, something to have in consideration?

etcd is built to be a simple, secure, and robust component. Key considerations for its architecture include:

* **High Availability:** For production environments, etcd is deployed as a **cluster of at least three members** to tolerate failures and maintain quorum. An odd number of members is recommended to prevent split-brain scenarios.
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

* **Leader Health:** Monitor the number of etcd leaders (`etcd_server_is_leader`) and a leader's uptime to ensure the leader is stable. Frequent leader changes could indicate network issues or instability.
* **Network and Replication Performance:** Track metrics related to peer communication and replication latency (`etcd_network_peer_sent_bytes_total`, `etcd_server_proposals_pending_total`). High latency can impact the entire cluster.
* **Read/Write Performance:** Monitor the latency of client requests (`etcd_request_duration_seconds`) to ensure that read and write operations are fast enough. High latency could be caused by slow disks or a high number of requests.
* **Disk I/O:** etcd's performance is heavily dependent on disk speed. Monitor disk write and sync duration (`etcd_disk_wal_fsync_duration_seconds`) to ensure it can keep up with the write load.
* **Resource Usage:** Monitor the CPU, memory, and disk usage of the etcd pods. If etcd runs out of resources, it can become unstable or lose its quorum.