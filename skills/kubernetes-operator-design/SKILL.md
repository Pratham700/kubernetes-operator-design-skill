---
name: kubernetes-operator-design
description: "Design, build, and optimize production-ready Kubernetes Operators in Go (controller-runtime), Helm, or Ansible. Adheres to CoreOS/Red Hat SRE principles, the Operator Maturity Model, and the Seven Habits of Highly Successful Operators. Integrates critical caching optimizations, OOMKill prevention, least-privilege RBAC scoping, finalizers, and OLM metadata packaging."
---

# Design Workflow for Production-Grade Kubernetes Operators

Follow this systematic workflow top to bottom whenever designing, implementing, refactoring, or reviewing a Kubernetes Operator.

## 1. Context Capture & Scoping
Before coding, record the following details:
- **Target environment criticality**: (dev/staging/prod)
- **Target cluster version floor** & distribution (e.g., vanilla K8s, EKS, OpenShift).
- **Target maturity level**: (Phase I: Install, Phase II: Upgrade, Phase III: Lifecycle, Phase IV: Insights, Phase V: Auto-Pilot).
- **Operator scope**: Namespace-scoped (isolated role bindings) or Cluster-scoped (cluster roles, watches across all namespaces).

## 2. API Design & Validation (CRD)
1. Model the custom resource schema carefully (see [reconciliation.md](file://references/reconciliation.md)).
2. Expose operational fields in the `.spec` (desired state) and status fields in `.status` (actual state).
3. **CRD Validation**: Write OpenAPI v3 schema validation for every property to reject malformed payloads before they are persisted in `etcd`.
4. Ensure schema backward compatibility (Habit 5).

## 3. Informer Cache Tuning (OOMKill Prevention)
Informer caches watch and cache resource types cluster-wide by default, which can lead to OOMKills. Implement cache isolation policies:
1. **Label Filtering**: Apply selectors to Informers (`cache.Options.ByObject`) to only cache resources managed by this operator.
2. **Strip Managed Fields**: Apply `cache.TransformStripManagedFields()` to strip `managedFields` and reduce memory footprint by up to 90%.
3. **OnlyMetadata Watches**: Use metadata-only watches (`WatchesMetadata` or `OnlyMetadata: true`) for high-volume resources.
4. **Avoid Cache Traps**: Never watch a resource as typed and read it as unstructured (or vice versa), which creates double caches.
5. **Disable Phantom Watches**: Set `ReaderFailOnMissingInformer: true` in manager options to crash-fast on implicit caches, and use `GetAPIReader()` for uncached, on-demand reads.

Refer to [cache-optimization.md](file://references/cache-optimization.md) for configurations and Go code templates.

## 4. Reconciliation Loop & Child Resource Lifecycle
1. **Ensure Idempotency**: Ensure that repeated reconciliation invocations do not duplicate work or child resources.
2. **Track Ownership**: Set controller owner references (`controllerutil.SetControllerReference`) for automatic cascade deletion by the garbage collector.
3. **Resource Naming**: Use dynamic, reproducible, namespace-scoped names (`<parent-name>-<suffix>`). Never hardcode.
4. **Finalizers**: Intercept deletions with a custom Finalizer to run pre-deletion cleanup (backups, deregistration).
5. **Decouple Lifecycles**: Ensure deleting the operator leaves the managed application running (Habit 4).

Refer to [reconciliation.md](file://references/reconciliation.md) for Go templates.

## 5. Security & Least-Privilege RBAC
1. **Scope Permissions**: Avoid wildcard (`*`) rules in Role/ClusterRoles.
2. **Restrict Verbs**: Grant only necessary verbs (`get`, `list`, `watch`, `create`, `update`, `patch`) on specific resources.
3. **Pod Security**: Harden the Operator deployment manifest (drop capabilities, runAsNonRoot, readOnlyRootFilesystem).

Refer to [rbac-security.md](file://references/rbac-security.md) for scoping templates.

## 6. OLM Packaging & Metrics
1. Generate the OLM bundle metadata skeleton via `operator-sdk`.
2. Populate the ClusterServiceVersion (CSV) with `specDescriptors` and `statusDescriptors` for UI integration.
3. Declare support for specific `installModes`.
4. Implement metrics to monitor the **Four Golden Signals**: Latency, Traffic, Errors, and Saturation.

Refer to [olm-packaging.md](file://references/olm-packaging.md) and [maturity-model.md](file://references/maturity-model.md) for OLM and telemetry specs.

---

## Reference Guides Index
- [Seven Habits of Successful Operators](file://references/seven-habits.md) — Core SRE philosophy.
- [Go Reconciler & Finalizers Code Guide](file://references/reconciliation.md) — Go code blocks for reconciliation and finalizers.
- [Informer Cache & OOMKill Prevention Guide](file://references/cache-optimization.md) — Mitigating memory vulnerability patterns.
- [RBAC Scoping & Security Hardening](file://references/rbac-security.md) — Fine-tuning operator permissions.
- [OLM Packaging & Metadata Specification](file://references/olm-packaging.md) — Designing OLM CSV and channels.
- [Operator Maturity Model & Golden Signals](file://references/maturity-model.md) — Progressing automation and instrumentation.
