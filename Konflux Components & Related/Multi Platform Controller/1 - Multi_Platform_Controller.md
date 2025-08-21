The Konflux Multi Platform Controller (MPC) is a specialized component within the Konflux ecosystem designed to facilitate the **native building of artifacts on various architectures without necessitating a multi-architecture OpenShift Container Platform (OCP) cluster**. It achieves this by provisioning virtual machines (VMs) on demand in cloud environments.

### About the Multi Platform Controller

The Multi Platform Controller is a core part of Konflux's infrastructure, enabling the execution of **remote build steps on remote VMs with diverse platforms**. It is integral for teams building applications that need to support multiple CPU architectures. For instance, the `build-vm-image.yaml` is an example task that leverages the Multi Platform Controller for provisioning VMs.

### Key Components

The functionality of the Multi Platform Controller relies on several key elements:

*   **Cloud VMs/Hosts**: The MPC utilizes **cloud-based virtual machines** to provide multi-architecture support. The range and capabilities of these VMs are directly dependent on what the underlying cloud providers offer. Different machine types, such as `linux-m2xlarge/arm664` and `linux-large/s390x`, are available to accommodate varying host sizes and resource requirements, replacing default smaller values.
*   **Tasks**: The MPC integrates with specific Tekton tasks, such as `buildah-remote` or `docker-build-multi-platform-oci-ta`. These tasks are designed to interact with the controller to execute builds on the provisioned VMs.
*   **Host Configuration (`host-config.yaml`)**: The available hosts and their specifications (like vCPUs and memory) for the Multi Platform Controller are defined in `host-config.yaml` files, typically found within the `infra-deployments` repository.
*   **SSH Secret Management**: For secure communication with the remote build VMs, the MPC handles the "SSH secret magic," which involves exchanging a one-time password (OTP) for the necessary SSH secret.
*   **OCI Artifacts**: Information is shared between tasks using **OCI artifacts instead of Persistent Volume Claims (PVCs)**, as PVCs can be problematic.

### How is Multi Platform Controller built, something to have in consideration?

The Multi Platform Controller is built to dynamically provision VMs for builds:

*   It leverages **cloud VMs** for its operations.
*   VMs are **spawned on demand**, which contributes to faster build times by avoiding delays associated with OCP cluster joining.
*   It supports a variety of platforms including **amd64, arm64, power, and z (s390x)** across different Konflux clusters. For example, the "p02" cluster supports all four, while "p01" and "rh01" support only amd64 and arm64.
*   Adding new AWS instance types is considered "very easy," often requiring only a **one-line commit**. However, enabling SSH for new operating systems like Windows and ensuring compatibility requires additional effort.
*   Configuration changes for the MPC, such as adding new VM types, involve modifications to files like `ec2.go` and `host-config`.
*   It is designed to avoid using Persistent Volume Claims (PVCs) by utilizing OCI artifacts for data sharing between tasks, recognizing that **PVCs can be problematic**.
*   There are considerations regarding **VM sizing**, with some platforms like s390x having what might be perceived as insufficient default resources (e.g., 4 vCPUs and 16 GiB of memory for a "large" platform). All variants typically have 40GB of storage.

### How does Multi Platform Controller work?

The Multi Platform Controller functions by:

*   **Executing build commands on platforms that match the target architecture**.
*   Integrating with **specific Konflux build pipelines**, such as `docker-build-multi-platform-oci-ta`, which orchestrate the build process.
*   Allowing users to **specify their desired build platforms** through parameters like `build-platforms` in their Component Custom Resources (CRs) or PipelineRuns.
*   Providing **root privileges** for build or test execution when needed, by supporting specific `linux-root` platforms (e.g., `linux-root/arm64` or `linux-root/amd64`).
*   Automating the **SSH secret exchange process** required to access the remote VMs where builds are performed.

### Normal Use Cases of the Multi Platform Controller

The Multi Platform Controller is essential for several key scenarios:

*   **Multi-Architecture Builds**: Its primary use case is to **build container images for multiple architectures** like x86_64, ppc, s390x, and aarch64, a common requirement for Red Hat products.
*   **File-Based Catalog (FBC) Builds**: While a dedicated "multi-platform FBC pipeline" is a requested feature, users currently adapt existing multi-platform pipelines to build FBCs for various architectures and OCP versions.
*   **Running Integration Tests in Privileged Environments**: It is used to execute tests that require specific privileges, such as **root access**, by selecting appropriate `linux-root` platforms.
*   **Resource-Intensive Builds**: For builds demanding more CPU or memory resources, the MPC can provision **larger VMs**, enabling high-performance builds that might otherwise face node resource issues in parallel build environments.
*   **Windows Container Builds**: Although challenging and requiring specialized expertise, the architecture of the Multi Platform Controller **theoretically supports building Windows containers** by provisioning Windows VMs, given sufficient effort to configure SSH and other Windows-specific aspects.

### Considerations for Monitoring Service Functionality

To ensure the Multi Platform Controller operates effectively, several monitoring aspects should be considered:

*   **Log Analysis**: It is crucial to examine the **Multi Platform Controller logs** for troubleshooting, especially when builds fail due to issues with platform provisioning. Uploading logs is always recommended for faster problem assessment.
*   **VM/Task Status**: Monitor the **number of running cloud tasks and VMs** to prevent resource exhaustion, as issues like "Too many running cloud tasks" can block builds.
*   **Configuration Updates**: Be aware of changes to the underlying `host-config.yaml` and updates to the controller itself, as these can impact the availability and performance of different platforms.
*   **Alerting**: The lack of specific alerts for controllers like the Multi Platform Controller can hinder proactive issue detection, highlighting a need for improved monitoring systems.
*   **JIRA Tracking**: Issues and feature requests related to the Multi Platform Controller are typically tracked in **JIRA projects**, such as `KFLUXINFRA` or `KONFLUX`.
*   **Documentation Clarity**: The evolving nature and occasional incompleteness of documentation can pose challenges for users trying to understand and troubleshoot MPC behavior. Regularly consulting and contributing to the official Konflux documentation is important.