### What is Kyverno?

Kyverno is an open-source policy engine designed specifically for **Kubernetes**. Its name comes from the Greek word for "govern." Kyverno allows you to manage, validate, mutate, and generate Kubernetes resources using policies written in standard YAML syntax, which is a key advantage as it eliminates the need to learn a new, domain-specific language like Rego. It operates as a dynamic admission controller within a Kubernetes cluster, intercepting API requests to ensure that configurations adhere to predefined rules and best practices.

***

### Key Components

Kyverno's architecture includes a few core components that work together to enforce policies:

* **Policy Engine:** The heart of Kyverno, this component is responsible for evaluating and enforcing policies on Kubernetes resources.
* **Admission Controllers:** Kyverno registers itself as both a **ValidatingAdmissionWebhook** and a **MutatingAdmissionWebhook**. These webhooks are what allow Kyverno to intercept API requests for creation, deletion, or modification of resources before they are saved to the cluster's etcd database.
* **Kyverno Policies (CRDs):** Policies in Kyverno are defined as Kubernetes Custom Resource Definitions (CRDs). This means they are managed just like any other Kubernetes object, making them easy to version control and apply with standard tools like `kubectl`. There are three main types of policies:
    * **Validate:** Checks if a resource is compliant with a policy and can either audit or block the request if it's non-compliant.
    * **Mutate:** Modifies a resource before it is created or updated to conform to a policy.
    * **Generate:** Automatically creates new resources based on the creation of another resource.
    * **Cleanup:** Deletes resources based on specified criteria.

***

### How is Kyverno built, something to have in consideration?

Kyverno is built as a single, lightweight Kubernetes application. It runs as a set of controllers within the cluster itself, leveraging the Kubernetes API for all its functionality. When considering how it's built and deployed, you should keep in mind that:

* **Native Integration:** Because it uses Kubernetes native resources and APIs, it integrates seamlessly with your existing workflows and tools.
* **Performance:** Kyverno processes admission requests in real-time. For large-scale clusters with many policies, you should consider setting appropriate resource requests and limits for the Kyverno pods and implementing pod topology spread constraints to distribute them across zones or nodes for high availability and performance.
* **Security:** As it's an admission controller, it has significant privileges. You should configure Role-Based Access Control (RBAC) to restrict its permissions to only what is necessary.

***

### How does Kyverno work?

Kyverno's workflow is tied to the Kubernetes API server's admission control process.

1.  A user or process sends a request to the Kubernetes API server to create, update, or delete a resource (e.g., a Pod or Deployment).
2.  After the request is authenticated and authorized, it is sent to the admission control stage.
3.  The Kubernetes API server, configured with Kyverno's webhooks, sends the resource to Kyverno for evaluation.
4.  Kyverno's policy engine checks the incoming resource against all matching policies:
    * If a **mutate** policy matches, Kyverno modifies the resource and sends the altered version back to the API server.
    * The (potentially modified) resource is then checked against **validate** policies. If a rule fails in "enforce" mode, the request is rejected with a descriptive error message. If it's in "audit" mode, a policy report is generated, and the request is allowed to proceed.
    * After the resource is persisted, **generate** policies can trigger, creating new resources.
5.  Based on the policy evaluation, the API server either accepts the request and persists the resource or rejects it.



***

### Which would be a normal use case for Kyverno?

A very common use case for Kyverno is to enforce **Pod Security Standards (PSS)**. For example, you can write a Kyverno policy that ensures all new Pods have a `runAsNonRoot` security context. Without this policy, developers might accidentally deploy containers that run as the `root` user, which is a major security risk.

Another use case is to enforce **labeling conventions**. For instance, you could require all Deployments to have a `team` label. A mutate policy could even automatically add a default `team` label if one is not present.

***

### What should I have in consideration to monitor to ensure functionality?

Monitoring Kyverno is crucial for ensuring its functionality and understanding its impact on your cluster. You should consider monitoring the following:

* **Policy Execution and Latency:** Monitor metrics like `kyverno_policy_execution_duration_seconds` to ensure policies aren't introducing significant latency into your API server.
* **Admission Review Latency:** Track `kyverno_admission_review_duration_seconds` to see the end-to-end time it takes for a request to be processed by Kyverno.
* **Policy Results:** Use `kyverno_policy_results_total` to track the number of successful (`pass`), failed (`fail`), and skipped (`skip`) policy evaluations. This helps you identify non-compliant resources and policy-related errors.
* **Controller Health:** Monitor the Kyverno pod status and logs for any errors or drops (`kyverno_controller_drop_count`), which could indicate unrecoverable issues.
* **Policy Changes:** Keep an eye on `kyverno_policy_changes_total` to track policy creations, updates, and deletions, which is vital for auditing and compliance.

You can use standard Kubernetes monitoring stacks like Prometheus and Grafana to collect and visualize these metrics, as Kyverno exposes them in a Prometheus-compatible format.

This video provides an introductory overview of Kyverno and its use cases for beginners.

[Kyverno for Beginners: Secure Your Kubernetes Clusters Like a Pro](https://www.youtube.com/watch?v=MxGAuVsJBXE)
http://googleusercontent.com/youtube_content/0