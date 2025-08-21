Etcd performance issues in Konflux can stem from a variety of factors, often related to resource exhaustion, high load, and specific configurations or bugs within the system.

Here are the key causes identified:

*   **Etcd Database Exceeding Capacity Limits**
    *   The most frequently reported issue is the etcd database running out of space, leading to "etcdserver: mvcc: database space exceeded" errors. This directly results in pipelines failing to start, new applications/components failing to be created, and even preventing existing resources from being deleted.
    *   **High Input Exceeding Processing Capacity**: The cluster's input load can be higher than what it can process, causing etcd to fill up.
    *   **Insufficient Cleanup/Pruning**: Inadequate or failing cleanup jobs for PipelineRuns and TaskRuns lead to their accumulation in etcd, putting significant stress on the database. This issue caused an outage where old PipelineRuns and TaskRuns (some 217 days old) could not be deleted normally, blocking the operator. The Tekton operator's pruner functionality was even reverted at one point due to hitting etcd limits.
    *   **Excessive Secret Mounting**: A high number of components (e.g., 600) associated with a single service account can lead to every pod mounting every `dockerconfigjson` secret, which puts "undue pressure on etcd" and is considered a leading cause of cluster API server outages.
    *   **High Concurrency of Jobs/Pipelines**: A large number of pipelines running simultaneously or being triggered in bursts can exacerbate etcd issues.
    *   **Lingering Finalizers**: PipelineRuns with lingering finalizers prevent their proper deletion, keeping them in etcd and contributing to the storage burden.
    *   **Failed Resource Creations/Deletions**: Frequent failures in creating or deleting resources, possibly due to quota limits, can generate numerous events that stress etcd.
    *   **Metrics Serving in Etcd-Shield**: A specific code change to enable serving metrics in the `etcd-shield` webhook caused "webhook request times to spike," directly impacting etcd performance.

*   **Resource Starvation of Core Components**
    *   **Tekton Controller Issues**: The Tekton controller, responsible for scheduling pods, can become **OOMKilled** (Out Of Memory Killed) or slow down due to insufficient memory or CPU resources, especially if it doesn't have explicit requests/limits set. A single replica holding all leases can also create a bottleneck. This directly impedes pod creation, which in turn affects API server interactions with etcd.
    *   **Other Critical Components OOMKilled**: Other controllers or services, such as the ArgoCD controller or Multi-Platform Controller (MPC), getting OOMKilled can lead to general cluster instability that indirectly impacts etcd performance.
    *   **Insufficient Task Resource Allocation**: If individual tasks within pipelines do not request sufficient CPU/memory, they can be OOMKilled or experience slow scheduling, adding to overall cluster load.

*   **Slow or Unresponsive External Services/Network Issues**
    *   **Quay.io Performance**: Slowness or intermittent unavailability of Quay.io, a container registry, can cause Tekton resolvers to time out while pulling image bundles or definitions, contributing to API pressure and potential etcd load.
    *   **Intermittent Network Failures**: General network problems or latency can affect communication with external registries or internal services.
    *   **DNS Degradation**: Issues with DNS resolution can prevent tasks from retrieving necessary resources, leading to timeouts and compounding performance problems.

*   **Architectural and Design Limitations**
    *   **Single-Node EBS Volume Dependency**: If a pipeline uses a workspace that is an EBS volume, all its tasks must run on the single node where that volume is attached. This can overload that specific node, impacting overall cluster health and implicitly etcd performance.
    *   **VM/Instance Concurrency Limits**: Reaching the maximum number of concurrent virtual machines or instances (e.g., 50 instances for build environments) causes a backlog of requests, increasing contention and potential etcd interactions as the system struggles to provision resources.
    *   **UI Querying Etcd for Historical Data**: The Konflux UI attempting to load historical PipelineRun and TaskRun data directly from etcd can contribute to performance issues, especially if the data is not being efficiently pruned and archived to an external database. This can result in slow UI loading times and further strain on etcd.

Overall, many etcd performance issues are interconnected, with one problem (like resource exhaustion or pruning failures) cascading to others, leading to a degraded cluster state. Konflux has implemented a temporary "etcd-shield" protection mechanism to block new PipelineRun creations when the cluster is overloaded, to prevent etcd from becoming completely full, while working on long-term solutions like pipeline queuing and optimizing storage usage.