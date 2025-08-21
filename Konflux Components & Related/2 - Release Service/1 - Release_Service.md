Konflux's Release Service is a critical component for delivering software products within the Red Hat ecosystem, acting as an **orchestration layer** that helps manage the entire release process from build artifacts to published advisories. It aims to streamline the pipeline for both managed services and self-managed products.

### The Release Service

The Release Service primarily acts as a **gateway** between development teams and the official Red Hat release mechanisms, effectively replacing parts of the traditional Errata Tool workflow. Its main responsibility is to take validated software artifacts (snapshots) and, after various checks and policy enforcements, facilitate their release to target destinations like `registry.redhat.io`. It also plays a key role in generating **advisory data** based on information provided in various release resources.

### Key Components of the Release Service

The functionality of the Release Service is centered around several custom resources and a dedicated catalog of pipelines:

*   **ReleasePlan (RP)**:
    *   A static custom resource that declares a **team's intention to release** content into a managed namespace.
    *   Defined in the **developer's tenant namespace**.
    *   Can be configured for **manual or automatic releases** using the `release.appstudio.openshift.io/auto-release` label. If `auto-release` is set to `true` on the ReleasePlan, a Release CR is automatically created when a build pipeline and its associated integration tests pass. If set to `false`, releases must be initiated manually.
    *   Can contain **version-specific information** for release notes, such as `synopsis`, `topic`, `description`, and `solution`.
*   **ReleasePlanAdmission (RPA)**:
    *   A Konflux custom resource that **supports a ReleasePlan** and contains **additional information** necessary for a release pipeline.
    *   Defined in a **managed workspace** (e.g., `rhtap-releng`), where secrets, service accounts, and other critical information are stored and access is restricted.
    *   Includes the **mapping of components to delivery repositories** and specifies details about container tagging.
    *   Must contain **product metadata** in its `spec.data.releaseNotes` section, including `product_id` (also known as `eng-id`), `product_name`, and `product_version`. These fields are typically propagated to the RPA when created on the cluster, preventing users from changing them easily.
    *   The `auto-release` label on an RPA has a different meaning than on an RP: if set to `false` on an RPA, it **disables releases entirely** via that RPA.
*   **Release (CR)**:
    *   Represents the **intent to release content *immediately*** with an advisory. When this resource is created, the release pipeline is triggered.
    *   Must contain **release-specific information** in its `spec.data.releaseNotes` section, such as the `type` of advisory (e.g., `RHBA`, `RHEA`, `RHSA`), a list of `issues` fixed, `cves` fixed with component mapping, and `references`.
    *   It is generally not recommended to put issues or CVEs in the ReleasePlanAdmission or ReleasePlan, as this would imply all releases via that plan fix the same issues/CVEs repeatedly; they should be in the Release CR.
*   **Release Service Catalog**:
    *   A **repository** (`konflux-ci/release-service-catalog`) that contains all the **managed pipelines** and **Tekton Tasks** provided by the Release Team.
    *   It maintains `development`, `staging`, and `production` branches. Changes are promoted weekly from `development` to `staging` (for e2e testing) and then to `production` (used by all users). Users are advised to use the `production` revision for stability.
*   **Snapshot**:
    *   A **list of images** for each component in an application, created automatically when any component is built.
    *   This snapshot is then used by the Release Service to generate a staging release or a final release.
    *   The Release Service releases the snapshot if it **passes Enterprise Contract (EC) policies**.
*   **Collectors**:
    *   Components that are **executed prior to managed release pipelines** during a release event.
    *   Currently, there are **JIRA collectors** (using JQL queries to gather issues) and **CVE collectors** (scanning commit messages for CVE identifiers). The goal is to automate the collection of on-prem Release metadata.
    *   Collected data, along with data in Release/RP/RPA, can be used with **Jinja2 templates** to render release notes (synopsis, topic, description, solution).

### How Release Service is Built

The Release Service is built on a GitOps model, primarily leveraging the `konflux-release-data` repository for configurations and the `release-service-catalog` for pipelines and tasks.

*   **GitOps-driven Configuration**: ReleasePlanAdmissions (RPAs) are configured and maintained in the `konflux-release-data` GitOps repository. This ensures version control and auditability for release configurations.
*   **Managed Pipelines**: The core release logic resides in pipelines within the `release-service-catalog` repository. These are Tekton pipelines that orchestrate steps like pushing images to registries, requesting signatures, and populating Pyxis (Red Hat's product content delivery system).
*   **Secrets and Service Accounts**: Release pipelines often require access to sensitive credentials (e.g., for pushing to official registries). These secrets and associated service accounts are stored in the managed workspace, separate from the developer's tenant workspace, and are referenced by the RPAs.
*   **Policy Enforcement (Enterprise Contract)**: A key aspect of the Release Service is its integration with Enterprise Contract policies. These policies act as **gates** that must be satisfied before a release can proceed. If policies are violated (e.g., missing required metadata, unaddressed CVEs), the release will fail.
*   **Templating for Dynamic Content**: The service supports Jinja2 templating for generating release notes content by combining static data from RPAs/RPs/Releases with dynamic data collected by collectors.
*   **No FBC Advisories**: Notably, FBC (File-Based Catalog) releases **do not create advisories**. If an advisory is desired, it would typically be for the whole OLM catalog, not just a specific product.

### How Build Service Works

The Build Service is responsible for compiling application source code into container images and other artifacts.

*   **Artifact Generation**: It builds applications and their components (e.g., operators, operands, bundles) as container images from their respective Git repositories.
*   **Triggering**: Builds are primarily triggered by specific Git events, such as **pull requests (pre-merge builds)** and **push events (post-merge builds)**. Manual triggers are also possible.
*   **Automated Checks**: During the build process, Konflux runs various checks against the images and their source repositories to ensure stability and security.
*   **Snapshot Creation**: Upon successful completion of a build pipeline, the Build Service (via the Integration Service) generates a **Snapshot**. This snapshot is a manifest of all built images for the application's components and is then consumed by the Release Service for potential release.
*   **Pipeline Definitions**: The tasks and capabilities of the build pipeline are driven by the `build-definitions` repository. Users can customize their build process by editing pipelines in their Git repositories.

### Normal Use Case of the Release Service

A typical use case involves product teams releasing their software for consumption by Red Hat customers or internal services.

*   **Product Releases to `registry.redhat.io`**: The primary use case is to release certified products to Red Hat's official container registry. This involves configuring RPAs in `konflux-release-data` and specifying the `rh-advisories` pipeline in the RPA to handle advisory creation and content publication.
*   **Staging Releases**: Teams often perform test releases to a staging registry (`registry.stage.redhat.io`) to validate the release process before a full production release.
*   **Managed vs. Self-Managed Releases**: Konflux aims to support both types of releases. While the same artifacts can be released to both, the OLM integration aspect has specific considerations.
*   **Automated Release Metadata**: The service is evolving to automatically collect metadata like JIRA issues and CVEs for advisories, reducing manual input.
*   **Multi-Stream and Multi-Version Releases**: Konflux can handle products supported on multiple OpenShift versions simultaneously (e.g., RHEL 8 and RHEL 9 images for different OCP versions), often requiring separate RPAs or applications per stream. Managing z-stream releases for bug fixes is also supported.
*   **Integration with Third-Party Tools**: While still evaluating, the goal is to allow interaction with tools like ServiceNow, Jira, and PagerDuty within the release workflow.
*   **Tenant Release Pipelines**: For scenarios where a development team wants to release software to destinations under their direct control, Konflux supports "tenant release pipelines" that run in the developer's namespace, using their own secrets.

### Monitoring and Ensuring Functionality

To monitor and ensure the functionality of the Release Service, several aspects need careful attention:

*   **Reviewing Release and Pipeline Logs**: This is the most crucial step for debugging failures. Users are consistently advised to upload logs before seeking help. The Release UI provides links to the underlying managed pipeline runs where detailed logs can be accessed.
*   **Understanding Release Status via UI**: The Konflux UI's "Releases" tab within an application provides a view of ongoing or previous releases. Checking the `PipelineRun` status is vital for immediate feedback on release outcomes.
*   **Enterprise Contract Policy Compliance**: A common cause of release failures is **Enterprise Contract policy violations**. Teams must ensure their release content adheres to policies like `registry-standard` or `app-interface-standard`. Errors related to missing required fields, unaddressed CVEs, or incorrect component names in JIRA trackers must be actively monitored and resolved.
*   **Release Notes and Advisory Content Validation**: The content provided in the `releaseNotes` section of the Release CR (especially `issues`, `cves`, `type`, `references`) must be correctly formatted and complete. Incorrect or missing information can cause pipeline failures. The `rh-advisories` pipeline is designed to be idempotent for the same content, preventing new advisories from being created on re-release unless the content changes.
*   **Product Security Metadata**: Ensuring that `product_id`, `product_name`, and `product_version` are correctly defined in the RPA, and that product security metadata files are in place, is essential to avoid "Missing Prodsec file" errors.
*   **Grace Period Configuration**: Be aware of the `release_period_days` (default 7 days) set in the ReleasePlan, which defines the grace period for releases.
*   **Distinction Between Release Targets**: Users must understand the distinction between tenant releases and managed releases. If both are desired, separate ReleasePlans and Releases are needed, as users are currently limited to one pipeline type per RPA.
*   **Staying Updated with Documentation**: Many users report confusion due to fragmented or outdated documentation. Regularly consulting and contributing to the official Konflux documentation (e.g., `konflux.pages.redhat.com/docs/users/releasing/`) and participating in user channels is crucial for staying informed about the evolving release process.