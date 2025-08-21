The Konflux Project Controller is a specialized component within the Konflux ecosystem designed to streamline the management of software projects, especially those with multiple versions and extensive CI/CD requirements.

### About the Project Controller

The Project Controller's primary purpose is to **simplify and automate the process of managing multiple software versions** in parallel, particularly when these versions are maintained using different Git branches. It aims to reduce the manual effort involved in defining separate **Components**, **Applications**, **ImageRepositories**, **IntegrationTestScenarios**, and **ReleasePlans** for each branch or collection of components that need to be tested and released together.

It is currently classified as **experimental**, meaning its API and support might change in the future. While no further development is immediately planned for it, it may eventually be replaced by direct multi-branch support integrated into Konflux Components.

### Key Components

The Project Controller introduces and leverages specific Konflux objects and concepts to achieve its automation goals:

*   **Project**: This object describes a major software endeavor that can involve multiple teams and span an extended period. A `Project` can encompass one or more development streams.
*   **ProjectDevelopmentStream (PDS)**: A `ProjectDevelopmentStream` signifies an independent line of development. Each PDS can contain one or more **Applications**, and each Application, in turn, can contain one or more **Components**.
*   **ProjectDevelopmentStreamTemplate (PDST)**: This is a core feature for automation. The PDST specifies the resources (like Applications and Components) that need to be created and allows for the use of **variables to customize them**. This enables the rapid creation of many similar development streams.
*   **Standard Konflux Resources**: The controller interacts with and manages existing Konflux resources such as **Applications**, **Components**, **ImageRepository**, **IntegrationTestScenario**, and **ReleasePlan** definitions.

### How is Project Controller built, something to have in consideration?

The Project Controller is designed to facilitate GitOps-driven resource management within Konflux, but it comes with specific architectural considerations:

*   **Direct Cluster Resource Creation**: When utilizing the Project Controller, it directly creates the necessary Konflux resources (like Applications and Components) on the cluster. This means **you won't need to maintain separate definition files for these resources within the `konflux-release-data` repository**.
*   **Skipped Pull Request Generation**: Unlike manual component onboarding, the Project Controller **does not automatically generate a Pull Request (PR) for the Pipeline-as-Code (PaC) pipeline definition**. It operates under the assumption that the Git branches you are using already contain these PaC pipeline files.
*   **Initial Onboarding Recommendation**: It is advised to **manually onboard at least one component from each repository** (e.g., through the Konflux UI) before attempting to manage additional components in that repository using the Project Controller. This initial step helps Konflux's Build Service set up necessary configurations that are difficult to replicate manually.
*   **Dynamic Nature**: Contrary to an earlier understanding, the Project Controller is a standard Kubernetes controller with a reconciliation loop. This means it **will actively modify resources if changes are detected in its templates or development streams**.
*   **Templating Scope Limitations**:
    *   It functions as a **version-oriented templating mechanism**, rather than a general-purpose one, meaning its application is primarily for managing different software versions.
    *   It **does not support templating the `data` field within `ReleasePlan` resources**.
    *   Specific fields within `ReleasePlan` that *are* supported for templating include `name`, `application`, and a particular `label` that links to the ReleasePlanAdmission.
    *   It currently **lacks support for templating the `spec.containerImage` field** and fields such as `git-provider` and `git-provider-url`, though the latter is under consideration for future addition.
*   **Community Installation**: The Project Controller is **not included by default in the community Konflux installation**. Users in such environments might need to install it separately, potentially via downstream kustomize directories, and a bug was suggested to address this gap.

### How does Project Controller work?

The Project Controller operates by integrating with your GitOps workflow to manage Konflux resources:

*   **GitOps-Driven Automation**: It works hand-in-hand with ArgoCD. When a project template is used, **ArgoCD synchronizes this template, and the Project Controller then processes it to create or update the relevant Konflux objects** on the cluster automatically.
*   **Streamlined Versioning**: By allowing the definition of templates with variables per version, the controller significantly **reduces the number of distinct configuration files needed** for multi-version products, simplifying maintenance.
*   **Automated Resource Provisioning**: It automates the provisioning of core Konflux resources such as Applications and Components based on the templates defined within the ProjectDevelopmentStreamTemplate.
*   **Integration Service Interaction**: The controller for the integration service monitors `Component` creation and deletion events. Issues can arise if components are created out of sequence, for instance, before their associated applications are established.
*   **No Release Management**: While adept at setting up build infrastructure, the Project Controller **does not handle the orchestration or management of product releases** themselves. This remains a separate, downstream process within Konflux.
*   **Limited IntegrationTestScenario Templating**: Currently, dynamic variables (like `{{.versionName}}`) are **not supported within the `context` names of `IntegrationTestScenario`** definitions.

### Normal Use Cases of the Project Controller

The Project Controller is designed for specific scenarios where managing a large number of related Konflux resources across multiple versions becomes complex:

*   **Managing Multi-Version Products**: Its primary and recommended use case is to **efficiently handle multiple concurrent versions of a software product**, especially when each version is developed and maintained on a separate Git branch.
*   **Automating CI/CD Setup**: It automates the setup of crucial CI/CD resources, including `ImageRepository`, `IntegrationTestScenario`, and `ReleasePlan` objects, for each component or application. This significantly **reduces the manual overhead** required to enable a full CI/CD pipeline.
*   **Scaling Component Management**: For large projects with numerous components (e.g., 40 or more) and multiple release streams, it helps manage the extensive set of `pipeline YAML` files and ensures **consistent synchronization of product-specific build steps**.
*   **Centralized Configuration Definition**: It facilitates defining Projects and ProjectDevelopmentStream resources via `tenants-config` files. This allows for a **centralized, GitOps-based approach** to defining the structural elements of your Konflux setup.
*   **Complex Environmental Requirements**: The controller's design, including its ability to handle configurations for multiple branches and versions, was influenced by complex needs such as those found in **CNV (OpenShift Virtualization)** environments.

### What should I have in consideration to monitor to ensure functionality of the controller?

While direct, explicit monitoring guidelines for the Project Controller are not extensively detailed in the provided sources, several aspects can be inferred for ensuring its proper functionality:

*   **Verify Resource Creation and Updates**: Regularly **check that Applications and Components are being created and updated** in the cluster as expected, based on your `ProjectDevelopmentStreamTemplate` configurations.
*   **Monitor Controller Logs**: In cases where resources are not being created or behave unexpectedly, **review the Project Controller's logs** for any errors or indications of issues. One user noted a problem where no errors were reported even when component creation failed, suggesting the need for thorough log analysis.
*   **Leverage Konflux JIRA for Tracking**: All feature requests, bugs, and issues related to the Project Controller are typically **tracked within the KONFLUX Jira project**. Users are encouraged to file new feature requests if existing capabilities do not meet their needs.
*   **Consult Documentation and Support Channels**: Given its "experimental" status and evolving nature, it's crucial to **frequently consult Konflux's official documentation** (e.g., `konflux-ci.dev/docs/advanced-how-tos/managing-multiple-versions/`) and engage with the Konflux support team (e.g., via the `#konflux-users` Slack channel) for troubleshooting assistance and updates.
*   **Acknowledge Alerting Gaps**: There's a general acknowledgment that **specific alerts for "this controller" might be lacking**, indicating an area where proactive monitoring could be improved.
*   **Understand Inter-Service Dependencies**: Be aware of how the Project Controller interacts with other Konflux services, such as the Integration Service. For instance, **issues may arise if components are created before their corresponding applications**.
*   **Be Mindful of Current Limitations**: Keep in mind the known limitations, such as the inability to template all resource fields (e.g., `spec.containerImage` or `ReleasePlan` data) or its behavior regarding initial PR generation, to anticipate and mitigate potential issues.