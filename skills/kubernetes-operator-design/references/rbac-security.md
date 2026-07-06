# RBAC Scoping & Security Hardening

To maintain safety in multi-tenant environments, Kubernetes Operators must be secured using least-privilege configurations. This guide details how to scope RBAC definitions and harden the Operator deployment manifest.

---

## 1. Avoid Wildcard Permissions (Security Anti-Pattern)
Many standard scaffolding tools generate overly permissive RBAC roles using wildcard (`*`) matching for resources and verbs. This poses a massive security risk in the event of an Operator compromise.

### Vulnerable RBAC Definition
```yaml
# Vulnerable: Allows any action on critical resource types cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-operator-role
rules:
- apiGroups: [""]
  resources: ["*"]  # Wildcard resource matching
  verbs: ["*"]      # Wildcard verb matching
- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["*"]
```

### Secured Least-Privilege RBAC Definition
Explicitly whitelist the exact API groups, resources, and verbs required.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role  # Use Role over ClusterRole where possible to isolate scope to a single namespace
metadata:
  namespace: my-operator-system
  name: my-operator-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "secrets", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["database.example.com"]
  resources: ["databases", "databases/status", "databases/finalizers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

---

## 2. Namespace-Scoped Roles vs. Cluster-Scoped Roles

When designing the Operator's integration, select the appropriate scope:

| Scope | Manifest Types | When to Use | Security Tradeoff |
| :--- | :--- | :--- | :--- |
| **Namespace-Scoped** | `Role` & `RoleBinding` | Operator only manages workloads in its own namespace or a pre-defined single namespace. | **Excellent**: Blast radius is completely isolated. A compromise cannot affect other namespaces. |
| **Cluster-Scoped** | `ClusterRole` & `ClusterRoleBinding` | Operator needs to watch resources across the entire cluster (e.g., Service Meshes, Cert managers, global databases). | **Risky**: Broad read/write access cluster-wide. Requires cache-filtering and label-selectors to reduce OOMKill vector. |

---

## 3. Deployment Security Context Hardening
Ensure that the Operator container is isolated at the pod OS-level using secure `securityContext` settings in the deployment manifest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-operator
  namespace: my-operator-system
spec:
  replicas: 1
  template:
    spec:
      securityContext:
        # Prevent running container processes as root
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: manager
        image: my-registry/my-operator:v1.0.0
        securityContext:
          # Prevent root escalation
          allowPrivilegeEscalation: false
          # Restrict write operations to the container root filesystem
          readOnlyRootFilesystem: true
          capabilities:
            # Drop all kernel privileges
            drop:
            - ALL
        resources:
          # CRITICAL: Always set resource requests and limits to prevent resource starvation
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```
