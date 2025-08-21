Konflux UI serves as the primary interface for users to interact with the Konflux system, enabling them to manage and monitor their software development and release processes [Previous turn, synthesis].

### Konflux UI Overview

The Konflux UI is an evolving web interface designed for Konflux users to interact with the platform. It provides a visual representation of applications, components, and pipelines [88, 502, Previous turn, synthesis]. Currently, **both an older UI and a newer UI are available**, though this is a temporary state, and user access between the two is not always reflected.

Users commonly report **significant performance issues with the UI**, including general slowness, unresponsiveness, or incorrect data displays [88, Previous turn, synthesis]. These problems severely impact the user experience, making it difficult to efficiently monitor and debug pipelines [Previous turn, synthesis]. There are ongoing efforts and Jira tickets specifically addressing **platform stability and performance optimizations**, including UI-related issues.

### Key Components

The Konflux UI interacts with various backend components to display information and manage operations:

*   **Tekton Results (Backend Database)**: A major cause of UI slowness stems from **slow queries to Tekton Results**, the backend database storing pipeline run and task run logs and metadata [Previous turn, synthesis]. Konflux uses an older, forked version of Tekton Results, contributing to display and log fetching issues [Previous turn, synthesis].
*   **Etcd**: Issues with `etcd` are a recurring problem, contributing to UI outages and general instability, potentially preventing access to tenants via both UI and CLI [184, Previous turn, synthesis].
*   **Renovate Bot (MintMaker)**: While MintMaker generates pull requests for updates, its pipeline runs are **not visible directly in the Konflux UI**, only through the OpenShift (OCP) UI or CLI.
*   **Konflux-console-frontend-build**: This repository is associated with building frontend applications, indicating its role in the UI's operational context.
*   **Artifacts Display**: A **prototype is in place to integrate artifact viewing directly into the Konflux UI**.

### How is Konflux UI Built/Considerations

The Konflux UI itself is a developed application, with its documentation largely written in **AsciiDoc format**. Contributors can suggest changes through a **web editor accessible directly from any Konflux documentation page** or by forking the GitHub repository.

Key considerations in its development and use include:

*   **Configuration as Code (GitOps)**: Konflux does **not automatically switch applications from UI configuration to code-based configuration**. Users must manually update their application's configuration by removing it from the UI and recreating a new configuration file in their repository. This implies that while the UI can initiate some configurations, comprehensive management often defaults to a GitOps approach.
*   **Design Flaws**: It has been argued that the UI has a "design flaw" by **allowing valid changes that can put the system into a broken state**.
*   **Monorepo Support**: The UI assumes that each new component is entirely separate, which may not align with how new versions are managed in a monorepo structure.
*   **Consoledot Frontend Applications**: For these specific applications, users must ensure `docker-build` is selected in the "Pipeline" field during creation in the UI.

### How Does Konflux UI Work

The Konflux UI provides an interface for interacting with various aspects of the CI/CD pipeline:

*   **Application and Component Management**: Users can **create applications and add components** by filling in repository information directly in the UI. However, editing existing component configurations (e.g., to add annotations) directly via the UI is not consistently supported; it is often recommended to manage these through Configuration as Code.
*   **Pipeline Execution and Monitoring**: The UI is intended to be used for **monitoring and debugging pipelines** [Previous turn, synthesis]. It displays pipeline runs, and users can filter pipeline run tables by type (build or test).
*   **Log Access**: The UI is meant to display logs for pipeline and task runs. However, users frequently encounter issues where **logs are unavailable, disappear quickly, or are incomplete** [Previous turn, synthesis]. The logs view can also be occasionally **blocked by other UI elements**. Accessing full logs often requires going to the underlying OpenShift (OCP) web UI.
*   **Data Display Issues**: The UI may sometimes display **incorrect or outdated information**, such as showing an old build as the latest or displaying incorrect PR statuses [Previous turn, synthesis].
*   **Functionality Gaps**: Features like the **"Rerun" and "Start new build" buttons often do not work reliably** [Previous turn, synthesis]. Similarly, editing or viewing secrets in the UI has been noted as problematic, with suggestions to use ExternalSecret CRs for managing secrets.
*   **Triggering Builds**: While the UI is the interface, builds are primarily **triggered by specific events like pull requests or push events**. Manual triggers are possible by submitting a merge/pull request or by a project maintainer adding `/ok-to-test`.

### Normal Use Cases for Konflux UI

*   **Monitoring and Debugging Pipelines**: This is a primary intended use, despite reported performance issues and log availability problems [Previous turn, synthesis].
*   **Creating and Onboarding Applications/Components**: Users initiate the setup of their projects by creating applications and adding components via the UI.
*   **Viewing Pipeline Runs and Logs**: Users navigate the UI to check the status of their builds and tests, and attempt to access associated logs.
*   **Managing Component Relationships (Nudges)**: The UI can be used to define component relationships, although issues with these features have been reported.
*   **Initiating Test Releases**: Users can attempt to conduct test releases through the UI.
*   **Retrieving SBOMs**: Users can utilize the UI to get SBOM (Software Bill of Materials) information and view installed RPMs.
*   **Ad-hoc Configuration**: For quick, non-GitOps-driven changes, users might attempt to modify configurations directly in the UI, although this can sometimes lead to system instability.

### What to Monitor to Ensure Functionality

To ensure the functionality of the Konflux UI and the underlying system it represents, users and support teams should monitor:

*   **UI Performance**: Watch for **general slowness, unresponsiveness, or prolonged page load times**, which can exceed 30 seconds [88, Previous turn, synthesis].
*   **Backend Database Performance**: Pay attention to **slow queries to Tekton Results** and issues with log availability (logs being unavailable, disappearing, or incomplete) [Previous turn, synthesis].
*   **Data Consistency**: Verify that the UI displays **correct and up-to-date information**, as it can sometimes show outdated or incorrect data [Previous turn, synthesis].
*   **Error Messages**: Look out for "Gateway Timeout" (504) or "Internal Server Error" (500/502) messages, which indicate broader platform issues impacting the UI [Previous turn, synthesis].
*   **Infrastructure Health**: Be aware of recurring **`etcd` issues** and general platform load, as these directly affect UI stability and overall system performance [184, Previous turn, synthesis].
*   **Resource Quotas**: Monitor if user namespaces are reaching their resource quotas, which can cause pipelines to fail or hang, affecting the perceived UI performance [Previous turn, synthesis].
*   **Feature Reliability**: Keep track of the reliability of specific UI functions, such as "Rerun" or "Start new build" buttons, and the ability to view/edit configurations or secrets [200, Previous turn, synthesis].
*   **Pipeline Run Visibility**: Ensure that all expected pipeline runs, including those triggered by Renovate, are visible either in the Konflux UI or the underlying OCP UI, to avoid missed activity.