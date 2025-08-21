### How Kueue Works

Kueue is a Kubernetes-native job queueing system designed to manage batch, HPC, and AI/ML workloads. Unlike a traditional scheduler that decides which pod goes on which node, Kueue acts as a higher-level controller that decides **when a job should be admitted** to the cluster based on available resources, quotas, and priority. It works in conjunction with the default Kubernetes scheduler, not as a replacement.

Kueue's core mechanism revolves around three main API objects:

1.  **Workload**: An internal Kueue object that acts as a representation of a batch job, such as a Kubernetes `Job` or a custom resource like a `RayJob`. When a new job is created, Kueue intercepts it and sets a `suspend` flag, preventing the job's pods from being scheduled immediately. It then creates a `Workload` object to manage this job.
2.  **LocalQueue**: A namespaced object that acts as a waiting area for workloads. Users submit their jobs to a `LocalQueue`, which then points to a `ClusterQueue`.
3.  **ClusterQueue**: A cluster-scoped object that holds the core scheduling logic and resource quotas. It defines a pool of resources (e.g., CPU, memory, GPUs) with specific limits. The `ClusterQueue` is where the queueing, prioritization, and preemption decisions are made.

When a job is submitted:

* It is placed in a `LocalQueue`.
* The Kueue controller watches the `LocalQueue` and, based on the `ClusterQueue`'s rules, decides which `Workload` to admit.
* The `ClusterQueue`'s rules consider factors like priority, resource quotas, and fairness policies (e.g., Fair Sharing, FIFO).
* When a `Workload` is selected, Kueue checks if the `ClusterQueue` has enough resources to satisfy the `Workload`'s requirements.
* If the resources are available, Kueue "un-suspends" the job, which then allows the default Kubernetes scheduler to create and place the pods on nodes.
* Once the job is running, Kueue tracks its resource consumption against the `ClusterQueue`'s quotas.

### Normal Use Case

A very common use case for Kueue is **managing a multi-tenant, GPU-accelerated cluster for AI/ML workloads.**

Imagine an organization with a shared Kubernetes cluster for multiple data science teams (e.g., Team A and Team B), each with a specific budget and resource requirements.

* **Problem:** Without Kueue, if Team A submits 10 large GPU-intensive jobs, they could monopolize all the available GPUs, preventing Team B's high-priority jobs from running, even if they arrive later. This leads to unfair resource allocation and potential project delays.
* **Solution with Kueue:**
    1.  **Admins define Quotas:** The cluster administrator creates a `ClusterQueue` that represents the total GPU capacity of the cluster. They then create a `LocalQueue` for each team, pointing to the `ClusterQueue`, and configure quotas to specify how many GPUs each team is allowed to use. For example, Team A might have a quota of 5 GPUs, and Team B might have a quota of 5 GPUs.
    2.  **Users Submit Jobs:** Team A submits their 10 jobs to their `LocalQueue`. Team B submits their high-priority job to their `LocalQueue`.
    3.  **Kueue Orchestrates:** Kueue's controller sees the jobs in both queues.
        * It admits Team A's first 5 jobs because they fit within their quota. The remaining 5 jobs for Team A remain in the queue, suspended.
        * When Team B's high-priority job arrives, Kueue's preemption or fair-sharing policies can kick in. Even if Team A is using all of its quota, Kueue can decide to preempt one of Team A's low-priority jobs to make room for Team B's high-priority job, ensuring that critical work gets done first.
    * **Gang Scheduling:** Kueue also supports **gang scheduling**, a crucial feature for distributed AI/ML workloads. It ensures that a job requiring multiple pods (e.g., a `RayJob` with a head and several workers) is only admitted if all required pods can be scheduled simultaneously. This prevents resource waste and deadlocks where a job partially starts but can never complete.

### Common Issues and Solutions

#### 1. Issues

* **Misconfigured Quotas**: If `ClusterQueue` or `LocalQueue` quotas are incorrectly set, jobs may never be admitted, or they may be admitted but fail to run because the underlying resources are insufficient. This is a common issue, as a small misconfiguration can have a cascading effect.
* **RBAC Permissions**: Users must have the correct Role-Based Access Control (**RBAC**) permissions to create and manage `LocalQueue` and `Workload` objects in their namespace. Without these, they cannot submit jobs to Kueue. Similarly, the Kueue controller itself needs broad permissions to manage jobs and other resources across the cluster.
* **Starvation**: In a cluster with many low-priority jobs and few resources, high-priority jobs might get stuck in the queue indefinitely if the low-priority jobs consume all the available quota and no preemption policy is configured.
* **Deadlocks**: While Kueue's gang scheduling is designed to prevent this, misconfigurations or interactions with other cluster components can lead to a deadlock where jobs are queued but never admitted because the available resources are fragmented in a way that no single job can be satisfied.

#### 2. Solutions

* **Quota Validation and Monitoring**: Use tools to validate the configurations of `ClusterQueue` and `LocalQueue` objects. Implement monitoring and alerting on `ClusterQueue` metrics to track resource usage and identify when queues are full or jobs are consistently pending. The Kueue community offers built-in Prometheus metrics for this purpose.
* **Regular Audits and Automation**: Regularly audit RBAC policies to ensure that users and service accounts have the minimum required permissions. Use a GitOps approach or similar automation to manage Kueue configurations, which ensures consistency and makes it easier to track changes and roll back misconfigurations.
* **Implement Preemption Policies**: Configure preemption within `ClusterQueue` to allow higher-priority jobs to preempt lower-priority ones. This is critical for preventing starvation and ensuring that time-sensitive workloads are executed.
* **Leverage AdmissionChecks**: Kueue's `AdmissionCheck` API can be used to integrate with external systems or cluster autoscalers. For example, if a job requires more resources than are currently available, Kueue can use an `AdmissionCheck` to trigger the cluster autoscaler to provision new nodes. Once the new nodes are ready, Kueue can then admit the job. This helps prevent jobs from being perpetually stuck in the queue waiting for resources.