Kubearchive is a critical component for the Konflux platform, primarily designed to manage and retain historical data, alleviate strain on the etcd database, and ensure the long-term accessibility of important records.

Here's a breakdown of its workings, key considerations, and use cases:

### How Kubearchive Works

Kubearchive's core function is to provide **long-term database support for Custom Resources (CRs)**, specifically **snapshots** and **releases**, by moving this data out of the etcd database. This directly addresses the critical issue of the etcd database "exceeding the 8GB maximum size" and experiencing "major outage".

It aims to:
*   **Prevent etcd saturation**: By offloading historical data, Kubearchive helps prevent etcd from becoming overloaded, which previously caused pipelines to fail and new resources to be blocked.
*   **Maintain data visibility**: Even after resources like releases and snapshots are **garbage collected (GC'd)** from the cluster's live state (and thus from etcd), Kubearchive ensures they **remain visible in the Konflux UI**.
*   **Replace Tekton Results**: It is intended to take over the role of storing "old stuff" like pipeline runs, which Tekton Results was initially responsible for.
*   **Enable historical operations**: It will allow for capabilities such as **initiating a release based on a snapshot CR that has already been garbage collected**.
*   **Mitigate cleanup issues**: It addresses problems where cleanup jobs, for instance, related to old PipelineRuns and TaskRuns, might fail due to "Out of Memory" (OOM) errors, preventing proper deletion and contributing to etcd bloat.

### Key Components

While the sources do not detail internal architectural components of Kubearchive itself, its functionality is deeply integrated with and impacts several core Konflux components and data types:
*   **Custom Resources (CRs)**: Specifically, **Snapshots** and **Releases** are the primary data types that Kubearchive is designed to archive and manage long-term.
*   **Etcd database**: Kubearchive directly interacts with etcd by moving data out of it to prevent performance issues and outages.
*   **Konflux UI**: Kubearchive ensures that archived historical data remains accessible and visible to users through the Konflux user interface.
*   **Tekton Results**: Kubearchive is designed to supersede Tekton Results for the storage of historical pipeline data.
*   **Integration Service**: This service creates snapshots after build pipelines successfully complete, and Kubearchive helps manage the lifecycle of these snapshots, especially when the integration service's garbage collector might not be working correctly.

### How it is Built or What to Have in Consideration

The sources indicate that Kubearchive is actively being integrated into Konflux, with an ongoing "process to get it installed". Although specific build details (e.g., programming language, internal architecture) are not provided, several considerations are highlighted:
*   **Addressing etcd capacity**: Its design must effectively move data to prevent the etcd database from exceeding its 8GB limit.
*   **Handling data volume**: It needs to manage a large number of historical records, given current snapshot retention counts (e.g., 600 non-PR, 70 PR snapshots) and issues with failed releases not being garbage collected.
*   **Data integrity and accessibility**: The archived data must remain consistent and fully retrievable for UI display and future operations, such as re-initiating releases from old snapshots.
*   **Mitigating race conditions**: There is a plan to "document the problem as a knowledge base article and recommend storing the manifests of snapshots off cluster to mitigate race conditions with GC". This suggests that how data is written to and read from Kubearchive needs to be robust against concurrent operations and garbage collection.
*   **Pruning functionality**: Its own pruning mechanisms need to be robust and reliable, as "bugs related to its pruning functionality" have been a concern.

### Normal Use Case for Kubearchive

The primary and normal use cases for Kubearchive revolve around **efficient and persistent data management** in Konflux:
*   **Archiving historical releases and snapshots**: It serves as the long-term repository for release and snapshot data, including associated logs, after they are no longer actively needed in the live cluster's etcd.
*   **Ensuring auditability and traceability**: By retaining historical records, Kubearchive allows users to look back at past pipeline runs, builds, and releases for auditing, debugging, or compliance purposes, even years later.
*   **Facilitating future releases**: The ability to initiate a new release from an archived snapshot (even if garbage collected from etcd) is a key feature.
*   **Maintaining Konflux UI performance**: By offloading historical data, it helps ensure the Konflux UI remains responsive, especially when querying historical PipelineRun and TaskRun data.
*   **Resolving etcd performance bottlenecks**: Fundamentally, its most important use case is to prevent and resolve the "major outage" and "slow performance" issues caused by an overloaded etcd database.

### What to Have in Consideration to Monitor

Monitoring Kubearchive is crucial for maintaining Konflux's stability and performance. Key areas to consider monitoring include:
*   **Etcd health and size**: Continuously monitor the **etcd database size** and its **performance metrics** to ensure it stays within operational limits and does not experience "database space exceeded" errors or "slow performance".
*   **Snapshot and Release quotas**: Monitor the **snapshot quota** for tenant namespaces and the number of existing snapshots to prevent limits from being hit, as this has historically prevented new snapshots from being created.
*   **Kubearchive's pruning and archival processes**: Ensure that Kubearchive's data offloading and pruning mechanisms are functioning correctly and that data is being moved out of etcd as intended. The resolution of "bugs related to its pruning functionality" indicates this is an area requiring attention.
*   **Garbage collection of failed releases**: Monitor for instances where "failed releases and associated snapshots not being garbage collected" to ensure Kubearchive (or associated cleanup mechanisms) is effectively processing these.
*   **Data accessibility and integrity**: Regularly verify that historical data stored in Kubearchive is accessible and correctly displayed in the Konflux UI, confirming that the archival process hasn't corrupted or lost data.
*   **Resource consumption**: Monitor Kubearchive's own resource consumption (CPU, memory, storage) to ensure it's operating efficiently and not becoming a new bottleneck. Pipeline runs themselves can be OOMKilled, and increasing memory limits has been a common troubleshooting step for other components.
*   **System logs**: Review logs for any errors or warnings related to Kubearchive operations, data migration, or interactions with etcd and the Konflux UI. Monitoring performance "with variance up to 5x between pipelineRuns" indicates the importance of granular logging and performance data.