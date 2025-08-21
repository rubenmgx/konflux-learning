Konflux users and teams frequently encounter a range of issues, primarily categorized around **Infrastructure & Platform Stability**, **Pipeline & Build Process**, and **User Experience & Observability**. Many of these issues are interconnected, with underlying infrastructure problems often manifesting as pipeline failures or UI unresponsiveness.

Here's a breakdown of common issues by their associated service or controller:

### Infrastructure & Platform Stability

These issues relate to the foundational components and overall health of the Konflux clusters.

*   **`etcd` Database (Backing Store for Kubernetes)**:
    *   **Capacity Exceeded**: The most critical and recurring issue is `etcd` reaching its storage capacity, resulting in `etcdserver: mvcc: database space exceeded` errors. This effectively stalls all new pipeline runs and can cause "cluster outage symptoms" [previous etcd response]. This is a "known issue" related to `etcd` exceeding its 8GB size limit [previous etcd response].
    *   **Repeated Overload**: The database frequently gets full again, making "compaction and defrag" operations necessary [previous etcd response, 58]. This is a "repeated pattern".
    *   **Impact on Services**: High `etcd` load has directly impacted other services, for instance, slowing down the Renovate bot schedule [previous etcd response].
*   **API Server (Kubernetes Control Plane Front-End)**:
    *   **Load and Unresponsiveness**: The API server experiences significant load issues and unresponsiveness, which can lead to CPU/memory starvation on master nodes and cause both `etcd` and the API server to go down [previous API server response]. This often results in 5xx errors (e.g., 500, 502, 503) [8, 171, previous API server response].
    *   **Forbidden Errors**: Users frequently encounter "Forbidden" (403) errors when trying to create or access resources due to insufficient permissions [previous API server response].
    *   **Rate Limiting**: API limits (e.g., 5,000 requests per hour for PaC) can be hit if too many `PipelineRuns` are triggered, causing issues [previous API server response]. Quay.io's rate limiting also causes "too many requests" errors [previous API server response].
*   **`etcd-shield` (Protection Mechanism for `etcd`)**:
    *   **Blocking Admissions**: `Etcd-shield` automatically activates to **block new `PipelineRun` admissions** when `etcd` approaches critical capacity, displaying messages like "PipelineRun admission currently not allowed" [previous etcd-shield response].
    *   **Unexpected Behaviors**: Despite its purpose, it has been associated with unexpected behaviors and bugs, including spikes in admission webhook durations (tracked by KFLUXINFRA-1885) [previous etcd-shield response].
    *   **Temporary Solution**: It is viewed as a temporary measure until a proper queuing feature for Tekton pipelines is implemented [previous etcd-shield response].
*   **External Cloud Providers (IBM Cloud, AWS)**:
    *   **VM Provisioning Failures**: Persistent issues with `IBM VMs` ending up in a failed state are due to "datacenter capabilities shortage" and "storage failures". This affects multi-platform provisioning and `s390x` builds.
    *   **Cloud Instability**: General "cloud instability" leads to platform issues.
    *   **Load Balancer Errors**: Cluster upgrades can cause Load Balancers to throw a large amount of 5xx errors.
*   **External Registries (`Quay.io`)**:
    *   **Instability and Failures**: `Quay.io` instability, including 502 errors, is a common reason for build and release failures.
    *   **Secret-related Failures**: Problems occur with `oras` not having the correct Quay secret or having too many, making it difficult to select the right one.
    *   **Read-only Mode**: `Quay.io` can enter a "read-only mode," causing issues.

### Pipeline & Build Process

These issues directly affect the execution and reliability of Konflux pipelines.

*   **Tekton Pipelines (General)**:
    *   **Flaky Runs**: Pipelines frequently fail intermittently for unclear reasons, often requiring manual re-runs. This includes errors like "multiple volumes with same name" and `context deadline exceeded`.
    *   **Task Definition Resolution**: Failures occur due to delays or inability to pull task definitions from bundle image repositories, tracked by `KFLUXBUGS-1784`.
    *   **Resource Exhaustion**: Container builds can crash due to "OOM or OOD" (Out Of Memory or Out Of Disk) issues.
    *   **Timeout Issues**: Pipelines experience timeouts, which can be related to insufficient infrastructure resources or specific external integrations like `Clamav`.
    *   **Admission Webhook Spikes**: `etcd-shield` is linked to unexpected spikes in admission webhook durations, even without corresponding logs from the shield itself [previous etcd-shield response].
*   **FBC (Full Bundle Catalog) Pipelines**:
    *   **Scale Challenges**: The `fbc-related-image-check` processes hundreds of images, increasing the likelihood of backend registry failures.
    *   **Version Management**: There's a lack of a supported way to build multiple FBC channels, and FBC does not currently support building multiple versions simultaneously.
    *   **Configuration Inconsistencies**: `validate-fbc` tasks can fail due to specific registry definitions or inconsistencies in handling metadata.
    *   **Operator Deployment**: The `deploy-fbc-operator` pipeline can fail if required CRDs like `ManifestWork` are not installed on the cluster.
*   **Integration Service (Responsible for Snapshots and ITS)**:
    *   **ITS Validation Delays**: Integration Test Scenario (ITS) validation can be delayed by controller crashes or cluster capacity problems.
    *   **Limited Control**: Users seek more granular control over ITS triggers, such as triggering only on specific component changes or rate-limiting runs.
*   **Enterprise Contract (EC) & Conforma (Policy Enforcement)**:
    *   **Flaky EC Checks**: EC checks frequently fail intermittently but often pass on re-trigger, causing frustration.
    *   **Non-isolated Reports**: EC reports are "not isolated to that component," meaning a single violation can cause an "overall test failure" for the entire pipeline, even if other components are fine.
    *   **Specific Violations**: Common EC violations include missing `CycloneDX SBOM`, `disallowed_inherited_labels`, and issues with "tampered tasks and missing tasks".
    *   **Policy Rule Management**: Users struggle with continuously updated Conforma rules that change from "Warning to Violation," leading to unexpected release blockers.
*   **Renovate Bot / Mintmaker (Dependency Updates)**:
    *   **Quota Saturation**: The Renovate bot can "immediately slam our namespace with way too much activity" by submitting many PRs simultaneously, leading to quota hits.
    *   **Frequent PRs**: Mintmaker opens PRs for Tekton updates "too frequently," causing rate-limiting situations.
    *   **Lack of Automation**: There's a desire for better "nudging support" across product teams for managing these updates, but no single Konflux ticket tracks this.
*   **Multi-Platform Controller (MPC)**:
    *   **Crashing/Failures**: The MPC experiences crashes and failures, contributing to build failures, particularly for `ppc64le` and `s390x` architectures.
*   **Cachi2 (Dependency Caching)**:
    *   **Interference Issues**: Cachi2 can cause problems when multiple package managers (yarn, npm, gomod) are enabled, sometimes leading to "fetching tags for other manager" issues.

### User Experience & Observability

These issues impact how users interact with Konflux and their ability to understand and troubleshoot problems.

*   **Lack of Visibility and Debugging Tools**:
    *   **Missing Logs**: Users frequently report that "log files being gone in an hour is a pretty large impediment to fixing failures". It can be difficult to access historical logs or see detailed error messages.
    *   **Limited Metrics**: There's a recognized need for better metrics (e.g., successful/failed resolutions for remote resolvers) and observability (e.g., `dashboard for warnings` from Conforma).
*   **User Interface (UI) Problems**:
    *   **Performance Issues**: The Konflux UI can have "incredibly long loading times" even for small workspaces and may be generally unresponsive or inaccessible.
    *   **Crashes**: The UI crashes when attempting to display "large EC reports".
    *   **Inconsistent Information**: The `Releases tab` may not show releases correctly due to label selector issues. The UI can also allow "valid changes" that put the system into a "broken state".
    *   **Missing Features**: Key features like configuring "nudges" are not available in the UI, and there are missing options for component setup in the "new UI".
*   **Documentation and Guidance**:
    *   **Terminology Confusion**: Users transitioning from CPaaS find Konflux terminology confusing and desire a "translator from cpaas to konflux".
    *   **Incomplete/Unclear Docs**: There's a need for clearer guidance on FBC adoption, common failure modes, and API usage [previous API server response].
    *   **Complex Workflows**: The process for managing multiple versions/applications via `ProjectDevelopmentStreamTemplate` is complex.
*   **Support Workflow**:
    *   The `konflux-users` channel often gets overwhelmed with "almost 150 issues created this week alone", and support threads are "not built for tracking long standing requests". Many issues are converted from support tickets to Jira bugs for proper tracking.