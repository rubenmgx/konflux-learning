
Konflux is a **cloud-native, secure build system and platform** that primarily aims to **automate the building, testing, and releasing of software projects and application components**. It is operated as a service, specifically focusing on consolidating internal Red Hat pipelines.

Key functions and features of Konflux include:
*   **Ensuring Stability, Security, and Conformance**: It manages build-time and integration tests using Tekton tasks and various tools like conftest, Clair for vulnerability scanning, and ClamAV for anti-virus checks.
*   **Flexible Builds**: It supports building many different types of software artifacts, including multi-architecture builds for arm64, amd64, ppc64le, and s390x.
*   **Automated Processes**: Konflux handles automated CVE remediation and facilitates publishing to `catalog.redhat.com`.
*   **Extensibility**: Users can customize and define PipelineDefinitions, and extend functionality by building new tasks.
*   **Secure Supply Chain**: It serves as a proving ground for ideas and tools in the Secure Supply Chain space, such as Enterprise Contract. Konflux CI aims to meet or exceed SLSA Build L3 (v1.0).

It is important to note that Konflux refers to the internal system Red Hat uses, while Red Hat Trusted Application Platform (RHTAP) is the customer-facing product.

The core services and controllers within Konflux include:

*   **Build Service**: This service is responsible for performing builds using Tekton PipelineRuns within a namespace.
It utilizes a ConfigMap to determine which Tekton pipelines are available for component configuration.
The Build Service also creates Repository Custom Resources (CRs) for components and handles the initial setup of pipeline configurations.
*   **Release Service**: This service manages the transfer of application ownership to external destinations, such as container registries. It can extract binaries from images at release time and publish them, for example, to GitHub releases API. The Release Service automatically creates a Release object when a component is built, provided a ReleasePlan exists for that application. Configuration for releases, including "On Prem" (product) and "Managed Services" (service) definitions, is stored in the `konflux-release-data` repository, which supports GitOps for onboarding various resources.
*   **Integration Service**: This service monitors for successful build pipeline completions and then composes a `Snapshot`. Integration tests, including Enterprise Contract (EC) validation, are executed against these snapshots. The Integration Service is also responsible for automatically initiating tests once a build is ready and handling the automatic cancellation of PipelineRuns. Issues with the `integration-service-controller-manager` can prevent snapshots from being created, affecting test execution and visibility in the UI.
*   **Multi-Platform Controller**: This controller enables the native building of artifacts for multiple architectures (such as arm64, amd64, ppc64le, s390x) without requiring a multi-arch OpenShift cluster. It queues builds and needs to account for varying VM sizes required for different multi-arch components.
*   **Project Controller**: Described as *experimental*, this controller helps in managing multiple product versions and automatically creating resources on the cluster, thereby reducing the need for manual setup. It can sometimes enter a "fighting" state.
*   **MintMaker**: This service handles **dependency management** and is built on top of **Renovate**. It automatically creates Merge Requests (MRs) or Pull Requests (PRs) for dependency updates, including RPM lock file updates. MintMaker runs by default every four hours and is enabled for each component. It effectively replaces Freshmaker for dependency updates.
*   **Pipelines as Code (PaC)**: This system integrates pipelines with Git and Git events, triggering builds based on pull requests or push events. Konflux supports PaC out-of-the-box.
*   **Tekton**: Konflux is fundamentally built upon Tekton (also known as OpenShift Pipelines).
    *   **Tekton Pipelines** and **Tasks** define the sequence of operations for builds and can be extended with custom functionality.
    *   The **TaskRun controller** manages events related to TaskRuns and their associated pods.
    *   **Tekton Results** is intended to store metadata and logs for PipelineRuns, though users have reported performance and accessibility issues with the UI displaying this data.
*   **Enterprise Contract (EC)**: This component enforces **SLSA conformity policies** on both build and release pipelines. It is part of the Red Hat Trusted Artifact Signer (RHTAS) and its validation is a key part of the integration tests.

Konflux also extends the **Kubernetes API** with its own custom resources, such as `Applications`, `Components`, `Snapshots`, and `EnterpriseContractPolicy`, which are defined and managed in a similar manner to built-in Kubernetes objects.

Additional components and concepts to consider:

*   **Quay.io**: This is the container registry Konflux uses to store built images, including ephemeral builds from pull requests and pushes. Konflux also pushes directly to specific branches in onboarded repositories on Quay.io.
*   **Snapshots**: These are resources that contain all the components in an application, and are used for integration tests and release processes. Snapshots are created for each individual component and can also be created for a group of components.
*   **ReleasePlan (RP)**: This Kubernetes object, defined in the user's namespace, specifies the release process for an application, with a target of "managed-workspace" for Konflux-managed pipelines. It links to the application and is used in the release process.
*   **ReleasePlanAdmission (RPA)**: This object, typically configured in the `konflux-release-data` repository, defines what needs to be released. It includes applications, policy, and a mapping of components to be released. It also specifies parameters like `releaseNotes.cves` and `releaseNotes.issues.fixed` for security tracking.
*   **Pyxis**: This is a system used for managing delivery repositories, allowing Konflux to create or update them. It interacts with the release process for content delivery.
*   **Comet**: Similar to Pyxis, Comet is another system used for managing delivery repositories. It's related to Red Hat's product catalog and where applications become available after being built and released in Konflux.
*   **Tekton Chains**: This system generates verifiable supply chain provenance for artifacts built by Tekton pipelines, providing authenticated metadata about how software artifacts were produced.
*   **Product Security Metadata / CPE ID**: These are crucial for tracking and ensuring compliance for products. CPE IDs are tied to RHEL versions and are part of the product definitions managed by ProdSec.
*   **Conftest**: This is a utility used for validating container information as part of Konflux's build-time tests.
*   **Clair / ClamAV**: These are tools integrated into Konflux for vulnerability scanning (Clair) and anti-virus scanning (ClamAV) as part of build-time security tests.
*   **Snyk / Coverity**: These are SAST (Static Application Security Testing) tools used for code scanning and identifying hardcoded passwords or other security issues in the build process.
*   **Renovate**: This bot is involved in dependency updates, often by creating PRs to update `.tekton` YAML files. It can be configured to ignore PRs against certain branches or manage updates.
*   **PNC (Product Naming Convention) / JBS (JVM Build Service)**: These are external systems relevant for Java/Middleware builds, particularly for handling "from-source" builds and fetching artifacts. Konflux is working towards better integration with these for Java components.
*   **Jira / ServiceNow / PagerDuty**: Konflux aims to interact with these external systems for various workflow stages, such as updating tickets on new builds or integrating with IT operations.
*   **Bonfire**: An internal tool used by integration tests to deploy components.
*   **ODCS (OpenShift Distribution for Containers)**: An integration that can be used to install Red Hat RPMs during the build process, potentially requiring builds to run in an internal cluster.