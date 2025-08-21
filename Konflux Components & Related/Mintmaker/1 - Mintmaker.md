The Konflux MintMaker is a crucial component within the Konflux ecosystem, primarily focused on **automating dependency management** and ensuring that software projects remain up-to-date and secure.

### About MintMaker

MintMaker is a **Konflux hosted instance of the Renovate bot**. Its core purpose is to **propose updates in your repository** by filing Pull Requests (PRs) or Merge Requests (MRs). This includes providing **important security fixes and general updates** that are vital for Konflux's operability. It is often referred to as **Konflux's Freshmaker** and is intended to evolve from being part of the build service into its own independent Renovate service. MintMaker supports detecting updates for various dependency types, with a primary focus on **parent image and RPM updates**. It is automatically enabled once a component is onboarded to Konflux. Unlike some other tools, MintMaker currently **does not offer a user interface** for direct interaction.

### Key Components

MintMaker's functionality relies on several key elements:

*   **Renovate**: MintMaker is fundamentally an instance of the Renovate bot, leveraging its capabilities for dependency management.
*   **`renovate.json`**: This configuration file, placed in the root of your repository, allows users to **customize MintMaker's behavior** and override its default settings.
*   **Konflux Components**: MintMaker operates by monitoring repositories associated with Konflux Components. For MintMaker to work, a **component needs to be created** and refer to the repository and branch.
*   **Konflux GitHub App**: The presence of the Konflux GitHub App installed in your organization or repository is a **prerequisite for MintMaker's operation**.
*   **RPM Lockfile Prototype (`rpm-lockfile-prototype`)**: This is a **custom extension** built into MintMaker that enables the automatic updating of RPM lockfiles. It relies on an input file, `rpms.in.yaml`.
*   **Renovate Presets**: MintMaker utilizes predefined Renovate presets, which offer common configuration options to simplify setup.
*   **Tekton Manager**: MintMaker includes specific managers for various package ecosystems, such as a **Tekton manager** that handles updates related to Tekton files.

### How MintMaker is built, something to have in consideration

MintMaker is a **Konflux component** and an **internally hosted instance of Renovate**. Here are key considerations about its implementation:

*   **Deployment and Versioning**: MintMaker is deployed using a **specific Git commit hash**, meaning there isn't a "stable" branch that users can directly include in their configurations. This ensures consistency in deployments.
*   **Configuration Overrides**: While MintMaker applies a **set of default options globally** to all repositories, users can **override these defaults** by placing a `renovate.json` file in the top-level directory of their repository.
*   **Minimal Onboarding Requirements**: A repository does not need to be "fully" onboarded to Konflux for MintMaker to function. Simply having a **component created for the repository** and the **Konflux GitHub app installed** is sufficient. Users can ignore the Konflux onboarding PRs that are generated.
*   **No Manual Triggering**: MintMaker operates on an **automatic schedule**; it is currently **not possible to trigger it manually**.
*   **Concurrency and PR Management**: MintMaker's architecture can lead to issues with **parallel Renovate runs**. When multiple instances run simultaneously, they may interpret PRs created by other instances as invalid and **autoclose them**. To prevent this, it is **recommended to set `pruneStaleBranches: false`** in your repository's `renovate.json` file. Modifying `branchConcurrentLimit` is generally not recommended due to the potential for old branches accumulating and blocking new PRs.
*   **RPM Lockfile Location**: For MintMaker to automatically update RPM lockfiles, the **`rpms.in.yaml` file must be located in the repository's root directory**. Support for `rpms.in.yaml` files in subdirectories is still under development or not currently supported.
*   **Subscription RPMs**: A notable limitation is that MintMaker currently **does not support RPMs that require a subscription**. This is a known issue, and a card (CWFHEALTH-3445) exists to track its resolution.
*   **Manager Configuration**: If you enable additional package managers in your `renovate.json`, you must **provide a complete list of all desired managers**. The `enabledManagers` configuration option in Renovate is not extensible between global and repository-level configurations.
*   **Conflict with Other Renovate Instances**: If another Renovate instance is enabled in your repository alongside MintMaker, especially with a similar configuration, it **can lead to unexpected issues and conflicts** due to both instances trying to use the same Git branches for updates.
*   **Pip-compile Manager**: The `pip-compile` manager within MintMaker is configured to use **Python 3.12** for `pip-tools` and cannot be invoked for other Python versions.

### How MintMaker works

MintMaker integrates with your Git repositories and Konflux components to provide automated dependency updates:

*   **Repository Monitoring**: It continuously **monitors Konflux Components** by fetching the repository URL and branch information. Based on this, it generates a server-side Renovate configuration.
*   **Configuration Merging**: This generated server-side configuration is **combined with any `renovate.json` file** found in your repository to determine the final update rules.
*   **Automated PR/MR Generation**: MintMaker automatically **creates PRs or MRs** in your repository when it detects available dependency updates.
*   **Scheduling**: MintMaker runs on a **regular schedule**. While it previously ran every 2 hours, it has since been adjusted to run **every 4 hours**. Different package managers also operate on specific weekly schedules to manage load (e.g., RPMs and lockfile maintenance daily, Dockerfiles on Monday, Git submodules on Tuesday, Go modules on Sunday).
*   **Security Updates**: Security-related updates are prioritized. They are flagged with a `[SECURITY]` suffix in the PR/MR title and **ignore any configured schedule**, ensuring they appear promptly.
*   **Automerge Behavior**: For MintMaker to automatically merge its generated PRs, your CI pipeline needs to be configured such that **all tests pass**, or you must explicitly set `ignoreTests: true` in your `renovate.json` configuration.
*   **RPM Lockfile Management**: MintMaker automatically **generates and updates RPM lockfiles** based on the `rpms.in.yaml` input file. It updates the entire lockfile rather than individual RPMs.
*   **Parent Image Updates**: It also creates PRs/MRs when **parent container images are updated**, regardless of whether the update is for a CVE fix.
*   **Branch Naming**: MintMaker follows a specific branch naming convention, typically using the prefix `konflux/mintmaker/{{ baseBranch }}` for its generated branches.
*   **Go Module Updates**: Support for Go dependency updates was recently enabled. However, there is a **known issue where `go.sum` files might not be updated correctly** alongside `go.mod` files.
*   **Nudging Distinction**: It is important to note that MintMaker is distinct from "nudging," which is a separate Renovate service managed by the build service. MintMaker **does not update PRs that are labeled `konflux-nudge`**.

### Normal Use Cases of MintMaker

MintMaker is designed for several key scenarios within the Konflux ecosystem:

*   **Keeping Konflux References Up-to-Date**: Its primary role is to ensure that all Konflux-related references within your repositories, such as base images, Tekton tasks, and other dependencies, are **automatically kept current**.
*   **Automated Dependency Management**: It automates the entire process of **managing software dependencies**, reducing manual effort and ensuring that projects use the latest, most secure versions.
*   **Security Fixes and Operability Updates**: MintMaker is crucial for **proposing important security fixes** (e.g., CVEs) and other updates that are essential for the continued operability and security of your Konflux-managed projects.
*   **Updating Base Images**: It handles the process of **updating base images** referenced within Dockerfiles, proposing necessary changes through PRs.
*   **Tekton Task Updates**: It updates **Tekton tasks**, ensuring that your CI/CD pipelines always use the latest definitions.
*   **RPM Lockfile Maintenance**: For projects that use RPMs, MintMaker automatically **updates RPM lockfiles**, which is vital for hermetic builds and dependency prefetching.
*   **Git Submodule Synchronization**: It **keeps Git submodules updated** with upstream changes, which is useful for projects that integrate external codebases this way.
*   **Multi-Version Product Management**: By updating dependencies across different Git branches (e.g., for different product versions), it helps in managing complex multi-version software products.
*   **Centralized Configuration**: It supports the use of **inherited configuration** (e.g., `org-inherited-config.json` in a `{{parentOrg}}/renovate-config` repository) to apply consistent Renovate settings across multiple repositories within the same organization.

### What should I have in consideration to monitor to ensure functionality of the service?

To ensure MintMaker is functioning correctly and to troubleshoot any issues, consider the following:

*   **MintMaker Dashboard (Splunk Logs)**: The primary way to monitor MintMaker's activity and troubleshoot problems is by checking its **logs, which are available on the MintMaker dashboard in Splunk** (`https://rhcorporate.splunkcloud.com/en-US/app/search/mintmaker_dashboard`). Access to the `mintmaker` namespace logs is typically required to view these.
*   **Scheduled Job Status**: Verify that the `create-dependencyupdatecheck` cronjob is present and running on its schedule (currently **every 4 hours**), and check the logs of its pods, as well as `renovate-job-xxxxx` pods, for any errors or failures indicating that Renovate jobs are not being created or completed successfully.
*   **PR Activity Monitoring**:
    *   **Check for Expected PRs**: Ensure that MintMaker is **creating PRs for dependency updates** as expected, including for all configured branches.
    *   **PR Title Consistency**: Be aware that MintMaker has been observed to sometimes **constantly change PR titles**, which could indicate internal inconsistencies.
    *   **Automerge Status**: If automerge is configured, ensure PRs are actually merging. If not, check if all CI tests are passing or if `ignoreTests: true` is properly set in your `renovate.json`.
    *   **Open PR Limit**: MintMaker has a default limit of **10 open PRs** at any given time. If this limit is reached, new PRs will not be created. This limit can be increased in your `renovate.json`.
*   **Configuration Validation**:
    *   **`renovate.json` Syntax**: Carefully validate the `renovate.json` file for any **JSON syntax errors** (e.g., trailing commas) or invalid schedule configurations, as these can prevent MintMaker from working.
    *   **`pruneStaleBranches`**: Ensure `pruneStaleBranches` is set to `false` in your `renovate.json` if you encounter issues with PRs being prematurely autoclosed due to parallel Renovate runs.
    *   **`enabledManagers`**: If you are enabling specific managers, ensure you provide a **complete list of all desired managers** in your `renovate.json`, as this configuration option does not merge between global and repository levels.
*   **Awareness of Known Limitations and Bugs**:
    *   **Out-of-Memory (OOM) Issues**: MintMaker has experienced **OOM crashes** leading to only a portion of components being processed in each cycle.
    *   **Subscription RPMs**: MintMaker currently **cannot update RPMs that require a subscription**.
    *   **Multiple Lockfiles in Subdirectories**: It **does not support updating multiple RPM lockfiles if they are located in subdirectories**; they must be in the repository root.
    *   **Go.sum Updates**: There is a known issue with `go.sum` files not being updated when `go.mod` files are.
    *   **Pip-compile Version**: The `pip-compile` manager only supports Python 3.12 due to how `pip-tools` is installed.
    *   **PR Author/Instance Clarity**: It can sometimes be **difficult to definitively determine which Renovate instance (e.g., MintMaker vs. another) opened a specific PR**.
    *   **Unpredictable Execution Order**: While MintMaker runs on a fixed schedule, the **order in which repositories are processed is unpredictable**, which might lead to delays in updates, as processing all repositories can take several hours.
*   **Utilize Konflux Jira and Documentation**: All feature requests, bugs, and issues related to MintMaker are tracked in the **CWFHEALTH Jira project**. Users are encouraged to consult the comprehensive MintMaker documentation within the Konflux user docs and the official Renovate documentation for detailed configuration options and troubleshooting.