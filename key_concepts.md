Here is a list of key concepts and their definitions, based on the provided sources:

*   **Application**
    *   In Konflux, an application is a **collection of components**.
    *   It describes a major piece of software that can be worked on by multiple teams over an extended period.
    *   An application encapsulates **integration tests for various components**.
    *   It is often mapped to the traditional release engineering concept of a "Product".
    *   A ProjectDevelopmentStream can contain one or more Applications.

*   **Build Pipeline**
    *   A **Tekton PipelineRun** that takes source code and transforms it into a **tested software artifact stored in a container registry**.
    *   The tasks in the build pipeline and their capabilities are currently driven by the `https://github.com/redhat-appstudio/build-definitions` repository.
    *   Konflux has separate pipelines for building and releasing artifacts.

*   **Build Pipeline Customization**
    *   The ability to **update and manage build pipelines for each component** within an application.

*   **Build-time Tests**
    *   A specific **TaskRun within a build pipeline** that does not convert source code into a software artifact.
    *   They ensure the **stability and security of the application**, its components, its build pipeline, and the testing environment.
    *   These tests are executed as Tekton tasks, often using `conftest` for validating container information.

*   **Component**
    *   In Konflux, a component generally maps to a **single build or container**.
    *   It corresponds to traditional release engineering concepts like a Brew package, DistGit repository, or Honey Badger component.
    *   An application can contain one or more components.

*   **Conforma (formerly Enterprise Contract)**
    *   A **coded policy that enforces what can and cannot be shipped**.
    *   It is a set of requirements imposed on software delivery artifacts, serving a gating role in allowing or preventing a release.
    *   The policy needs to pass for any Snapshot before a release pipeline can proceed.
    *   Conforma requires that all Konflux pipelines use only tasks with recorded provenance in the trusted Task list.

*   **Container Verification Pipeline (CVP)**
    *   A service that **runs in the internal pipeline to trigger multiple types of container tests**.
    *   It supports "Functional Tests" which are developed by teams and run inside CVP's container pipelines.
    *   CVP provides Ansible variables (properties) that are passed to a playbook to kick off team-written tests.

*   **Custom Task**
    *   A user can **define a task directly in the pipeline file** using `taskSpec`, rather than using `taskRef`.
    *   You can create your own tasks in Git or as a Tekton bundle and use a resolver in the Pipeline Definition.
    *   To be able to add a custom task, you might need to use Trusted Artifacts (TA).
    *   They are implemented with an `apiVersion` that does not have a `tekton.dev/` prefix.

*   **Integration Tests (ITS)**
    *   Associated with testing across multiple components.
    *   The integration service creates the PipelineRun and references the pipeline definition for integration tests.
    *   IntegrationTestScenario resources are registered for an application.
    *   IntegrationTestScenario currently **does not support workspaces directly**, but a workspace can be defined in the pipeline definition that the IntegrationTestScenario points to. A spike (STONEINTG-895) is investigating providing extra attributes including workspaces for ITS.
    *   They are generally believed to run in users' workspaces.

*   **Namespace**
    *   In a Kubernetes cluster, it can have **sole or shared ownership**.
    *   It is the most global resource generally available to a user.
    *   Most Kubernetes Custom Resources (CRs) like Components, Applications, Snapshots, and Secrets are scoped to a single namespace.
    *   All Tekton PipelineRuns are performed within a namespace.

*   **Object**
    *   A **Kubernetes API object**.
    *   It can refer to a built-in, core Kubernetes object (like a Pod) or a Custom Resource Definition (CRD).

*   **Pipeline**
    *   A **collection of Tasks executed in a specific order**.
    *   It defines the tasks and their execution flow within a PipelineRun.
    *   Konflux provides multiple Tekton pipeline bundles in `quay.io/konflux-ci` for different use cases.

*   **PipelineRun**
    *   A process that **executes a Pipeline on a cluster with inputs, outputs, and execution parameters**.
    *   Konflux creates PipelineRuns in response to pull request and push events in your repository.
    *   A PipelineRun contains TaskRuns, and each TaskRun corresponds to one pod.
    *   The maximum values for CPU and memory for tasks are defined in the PipelineRun, as long as they are within the resource quotas.
    *   PipelineRun timeout takes precedence over an individual task's timeout.

*   **Pipelines as Code (PaC)**
    *   A practice that **defines pipelines using source code in Git**.
    *   It is also the name of a subsystem that executes those pipelines.
    *   PaC parses `.yaml` or `.yml` files in the `.tekton` directory and subdirectories at the root of your repository to detect Tekton resources like Pipeline, PipelineRun, and Task.
    *   It supports **dynamic variables** (`{{ var }}`) for use in templates, such as `revision` (commit SHA) and `repo_url` (repository URL).
    *   PaC automatically inlines remote tasks or pipelines referenced in a PipelineRun or PipelineSpec.

*   **Provenance**
    *   Metadata describing **where, when, and how the associated software artifact was produced**.
    *   `pipelinerun-provenance` is available under the hood in Konflux but not yet exposed as a user-facing feature.

*   **Project**
    *   A major piece of software that can be worked on by multiple teams over an extended period of time.
    *   A project may contain one or more development streams.

*   **ProjectDevelopmentStream**
    *   Indicates an **independent stream of development**.
    *   It can contain one or more Applications.

*   **Release Pipeline**
    *   A **generic Tekton pipeline that moves artifacts built within Konflux to somewhere outside of its control**.
    *   An application snapshot must pass the Conforma Policy check before Konflux can run the release pipeline.
    *   Managed release pipelines are supplied by the Release team and stored in the **Release Service Catalog** (`https://github.com/konflux-ci/release-service-catalog`).
    *   Tenant pipelines are user-defined pipelines.

*   **Release Plan (RP)**
    *   A **declaration of intent to ship specified content from a developer workspace to a managed (our) workspace**.
    *   A static custom resource that links your application to a ReleasePlanAdmission and optionally a tenant or final pipeline.
    *   A Release Plan is for the whole application and cannot exclude some components.

*   **Release Plan Admission (RPA)**
    *   The **business logic of the release process**.
    *   It defines the release process in the form of a Tekton Pipeline (or reference to a pipeline).
    *   An RPA CR exists within a managed workspace.
    *   It defines the specific pipeline to run and the Conforma Policy that must pass for a Snapshot before the pipeline can proceed.
    *   RPA also declares content-to-destination mapping and other parameters needed to execute the pipeline.
    *   The ReleasePlanAdmission will automatically pass parameters `taskGitRevision` and `taskGitUrl` to the pipeline, so you should not pass your own values for these.

*   **Software Bill of Materials (SBOM)**
    *   Currently, Konflux uses **CycloneDX format** for SBOMs, though migration to SPDX is likely planned.
    *   The `buildah` task is responsible for generating and pushing SBOMs.

*   **Task**
    *   **One or more steps that run container images**, where each performs a piece of construction work.
    *   Tasks can be stored with the repository being onboarded to Konflux.
    *   There is no single definitive repository for all tasks in Konflux, but common ones include `https://github.com/konflux-ci/build-definitions/tree/main/task`, `https://github.com/tektoncd/catalog/tree/main/task`, and `https://github.com/konflux-ci/konflux-qe-definitions`.

*   **TaskRun**
    *   A process that **executes a Task on a cluster** with inputs, outputs, and execution parameters.
    *   Konflux creates TaskRuns as part of a PipelineRun (each Task in the Pipeline results in a TaskRun).
    *   TaskRuns have default resource requests and limits, which can be overridden in the Task definition.
    *   If a TaskRun is stuck in pending and `tkn` cannot fetch logs, text is stored to S3.

*   **Tekton**
    *   A **Knative-based framework for CI/CD pipelines**.
    *   It is decoupled, allowing deployment to any Kubernetes cluster across multiple hybrid cloud providers.
    *   Tekton stores everything related to a pipeline in the cluster.
    *   Tekton objects (Tasks, TaskRuns, Pipelines, PipelineRuns) are implemented as Custom Resource Definitions (CRDs) and use validating admission webhooks.

*   **Tekton Bundle**
    *   A way to **package and share Tekton Tasks and Pipelines**.
    *   Tasks can be pushed to an image repository using `tkn bundle` and added to a ConfigMap for availability.
    *   Conforma requires that tasks are defined in a set of known and trusted task bundles.

*   **Tekton Workspace**
    *   A **storage volume that a task requires at runtime** to receive input or provide output.
    *   Required workspaces are defined in a Tekton PipelineRun.
    *   The `/workspace` directory is where volumes for resources and workspaces are mounted.

*   **Tenant Namespace**
    *   A namespace where **artifacts are produced from Tekton Pipelines**.
    *   These can be accessed by more than one individual according to defined roles and permissions.
    *   Tenant namespaces can be for an individual or a team.

*   **Trusted Task**
    *   A task that has its **unique reference (image digest or git commit ID) listed in the Trusted Task list**.
    *   Konflux uses trusted tasks to ensure secure and compliant task execution in build pipelines.
    *   Submitting a task to the `build-definitions` repository practically marks it as "trusted" by Red Hat.

*   **Workspace**
    *   A **Kubernetes namespace owned by an individual or group**.
    *   All Tekton Pipelines (build, test, release) run within a workspace.
    *   Users have access to at least one workspace and can be granted access to more in three tiers: Contributor, Maintainer, and Admin.
    *   Workspace separation is usually team-based.