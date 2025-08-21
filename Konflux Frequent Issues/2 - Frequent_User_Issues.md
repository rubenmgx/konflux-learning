Konflux users experience a range of issues, from general platform instability and performance degradation to specific pipeline failures and user interface challenges. These problems often stem from the complexity of integrating various systems and the need for robust stability in a multi-tenant environment.

Here are the most common issues users face:

### UI & User Experience Issues
*   **Slowness and Unresponsiveness**
    *   Users frequently report that the Konflux UI is **very slow**, especially for "activity" and "components" tabs, often taking **tens of seconds to minutes to load** or sometimes not loading at all, displaying a spinning blue circle.
    *   This slowness can lead to **"Gateway Timeout" (504) or "Internal Server Error" (500/502) messages**, making the UI difficult or even unusable.
    *   The UI can also be **inconsistent across different browsers** (e.g., working in Chrome but not Safari).
*   **Access and Permissions Problems**
    *   Users report **difficulty logging in** or **accessing their applications**, sometimes requiring frequent re-logins.
    *   Issues occur with **granting user permissions**, particularly for users with a `@redhat.com` login, as the UI may prevent them from being added to access control lists manually.
    *   User access provided from the old UI **does not reflect in the new UI and vice versa**.
    *   Users have experienced **lost workspaces** in the new Konflux console.
*   **Inconsistent Information Display**
    *   The UI may display **incorrect or outdated information**, such as pipeline statuses (e.g., showing "Merge pull request" without an actual MR/PR, or a release pipeline as failed when it succeeded).
    *   There can be a **mismatch between Konflux UI data and release pipeline data**.
*   **Limited UI Functionality and Clarity**
    *   Users desire an **"in UI issues dashboard"** to highlight problems and solutions.
    *   It's challenging to **edit non-working component configurations** directly via the UI, with users finding "zero UI ways to edit".
    *   The UI can allow "valid changes" that **put the system into a "broken state,"** which is considered a design flaw.
    *   The **logs view in the UI is often blocked** by elements or is incomplete/unavailable, significantly hindering debugging efforts.
    *   Users struggle to trigger existing or new builds from the Konflux UI.

### Pipeline & Build Process Issues
*   **General Build Failures & Instability**
    *   **Random and intermittent pipeline failures** are a frequent complaint, sometimes tied to Konflux infrastructure issues or external factors like Quay.io instability.
    *   Pipelines frequently become **stalled, pending, or stuck**, preventing completion of tasks.
    *   Builds can **take excessively long or time out**, varying significantly with server usage.
*   **Resource and Quota Limitations**
    *   Pipelines frequently **hit resource quotas** when many changes are introduced simultaneously, or when numerous pipelines are running.
    *   Pipelines can **silently crash or time out** without clear error messages if too much RAM/CPU is requested, with no pre-build validation to flag misconfigurations.
    *   There is a recognized **need for more disk space** on `amd64` and `arm64` instances, as it blocks image onboarding.
*   **Dependency Management (Cachi2)**
    *   The `prefetch-dependencies` step is **intermittently killed or fails**, especially for large RPMs (1+ GB) or when encountering `context deadline exceeded` errors.
    *   Cachi2 jobs can **crash unexpectedly**, particularly with Node.js dependencies, lacking clear debug information.
    *   Problems occur with **downloading Go dependencies** for SAST programs, even when the build task itself succeeds locally.
*   **Git and Repository Interactions**
    *   Issues with `git clone` operations, sometimes due to **tokens not being found**.
    *   **Pull requests can get stuck** or fail to reach their destination, with users asking how to investigate hanging "sending pull request" processes.
    *   Konflux's `on-pull-request` pipelines can be **triggered multiple times** unintentionally.
    *   The `konflux-redhat-bot` creates PRs where certain **checks are not triggered**.
    *   Users encounter **problems talking to GitHub** or intermittent `gitlab.cee.redhat.com` connection failures.
*   **Specific Task Failures**
    *   Tasks like `sast-shell-check` frequently encounter **timeout issues**.
    *   The `create-advisory` release task often fails.
    *   The `ClamAV` integration is prone to **timeouts**, blocking releases.
    *   The `add-fbc-contribution-to-index-image` task fails during migration.
    *   The `verify-conforma` task has been observed to be **OOMKilled** in staging releases.

### Enterprise Contract (EC) & Policy Enforcement Issues
*   **Constantly Changing Rules:** A significant pain point is that **Conforma rules are continuously updated and can change from "Warning to Violation,"** which leads to unexpected release blockers. Users desire a consistently updated document listing these rule changes and their effective dates.
*   **Non-Isolated Reports:** EC reports are "not isolated to that component," meaning a single violation in one component can cause an **"overall test failure"** for the entire pipeline, even if other components are compliant.
*   **Common Violations:** Users encounter violations such as **missing `CycloneDX SBOM`**, `olm.unmapped_references`, `disallowed_inherited_labels` [previous response], and issues with "tampered tasks and missing tasks" [previous response].
*   **False Positives in Scans:** **SAST Snyk emits "many false positives,"** and users struggle to configure Snyk to exclude these [previous response]. Similarly, **Clair scan produces false positives** and lacks a clear mechanism to "waive" them.
*   **Limited Debugging for EC Failures:** Users report **inability to see detailed logs for SAST Snyk failures** [previous response] and general difficulty viewing logs for failed EC checks in the UI.
*   EC pipeline failures are a common occurrence.

### Renovate Bot / Mintmaker Issues
*   **Excessive PR Generation:** The Renovate bot can "immediately slam our namespace with way too much activity" by submitting many PRs simultaneously. Mintmaker opens PRs for Tekton updates "too frequently," causing **rate-limiting and dependency dashboard issues**.
*   **Manual Intervention Required:** Many Renovate PRs **won't rebase automatically**, requiring manual checkout and merge operations.
*   **Lack of Automation/Nudging Support:** Users desire better "nudging support" across product teams for managing these updates, and a "cross application" nudge feature is a known gap in the UI.
*   **Continuous Updates:** Users express concern over "Konflux reference update" MRs that "exist at all and require regular monitoring/work," adding new work specific to Konflux.
*   Sometimes, **no new Konflux updates are opened** by Mintmaker.
*   The bot may propose PRs to **revert hermetic build settings**.

### FBC (Full Bundle Catalog) & Operator Release Issues
*   **Mandatory Requirement:** FBC is a requirement when publishing operators on Konflux.
*   **Migration Challenges:** Users transitioning from CPaaS find the FBC approach difficult due to high cadence of `z`-stream releases and differences between CPaaS Bundle images and FBC images, leading to **concerns about behavior discrepancies**.
*   **Limited Multi-Version Support:** There is **no supported way to build multiple FBC channels**, and FBC does not currently support building multiple versions simultaneously within a single Konflux application using a matrix configuration.
*   **Deployment Dependencies:** The `deploy-fbc-operator` pipeline can fail if required Custom Resource Definitions (CRDs) like `ManifestWork` are not installed on the cluster.
*   **Preflight Limitations:** Preflight does not support operator bundles in Konflux.
*   **FBC Pipeline Failures:** Issues like `add-fbc-contribution-to-index-image` failing and multi-component FBCs potentially not working or having their artifacts handled improperly.

### Infrastructure & Platform Stability
*   **External Registry Instability:** Beyond general instability, specific issues with `Quay.io` include **"bad gateway" errors, the registry entering "read-only mode," and 502 errors**. There is a recognized need for better telemetry for image pull/push to proactively escalate issues to the Quay team.
*   **Cloud Provider Issues:** Persistent issues occur with **IBM Virtual Machines (VMs) ending up in a failed state** due to "datacenter capabilities shortage" and "storage failures," particularly impacting multi-platform provisioning for `s390x` and `ppc64le` architectures.
*   **Service Instability:** The `integration-service` has been observed in a `CrashLoopBackOff` state on production clusters, although a bug fix was reported.
*   **Intermittent Connectivity:** Users experience **intermittent 502 errors** from the Konflux UI and frequent failures to connect to internal GitLab instances.
*   **Subscription Manager Issues:** Frequent "subscription-manager error" messages, especially on `s390x` builds, indicate problems with the subscription server, causing significant outages and preventing builds.

### Documentation, Guidance & Support Process
*   **Incomplete/Unclear Documentation:** Users struggle to find comprehensive guides for solutions, such as configuring custom tasks, understanding `konfig.yaml`, or finding spec references for various Custom Resource Versions (CRVs).
*   **Onboarding Challenges:** There's a need for clearer guidance on FBC adoption and managing upstream/public vs. downstream/private projects, as well as how Konflux handles subscriptions.
*   **Support Workflow Issues:**
    *   Support tickets are perceived as being **"closed quickly,"** hindering ongoing conversations about complex issues. Users are consistently asked to open new support workflow threads for each distinct question, even if related, which can be an "overhead" [previous response].
    *   There is a strong desire for a **central user community channel** for discussions, best practices, and workarounds, as the main `konflux-users` channel is gated by a support workflow.
    *   Users face difficulty in getting detailed logs, with reports of **log files disappearing within an hour**, making debugging challenging.
    *   There is a general request for **better debugging information** across all Konflux services.
    *   Users also suggest improving error messages in the UI for common failures to reduce the need for support requests.
*   **Lack of Status Page:** Users request a dedicated **status page or dashboard** to monitor the availability of the Konflux system.