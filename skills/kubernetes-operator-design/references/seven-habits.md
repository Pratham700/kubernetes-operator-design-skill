# The Seven Habits of Highly Successful Operators

Based on CoreOS/Red Hat design guidelines, follow these seven architectural habits to ensure your Operator runs reliably, behaves as a true "software SRE", and integrates with the Kubernetes ecosystem.

---

## 1. Run as a Single Kubernetes Deployment
An Operator must run as a controller inside a single-replica Kubernetes `Deployment`. 
- **Rationale**: The control plane manager must have a single active reconciler instance (or use leader election for active-passive hot standbys) to prevent race conditions when updating custom resource statuses.
- **Scaffolding Example**:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-operator
    namespace: my-operator-system
  spec:
    replicas: 1  # Always start with 1 replica; utilize leader election for HA
    selector:
      matchLabels:
        control-plane: controller-manager
    template:
      metadata:
        labels:
          control-plane: controller-manager
      spec:
        containers:
        - name: manager
          image: my-registry/my-operator:v1.0.0
  ```

---

## 2. Define New Custom Resource Types (CRDs)
Every Operator must declare its API using Custom Resource Definitions (CRDs). The custom resource represents the desired state of the managed application (the operand).
- **Rationale**: Declaring CRDs extends the Kubernetes API, allowing developers to interact with your application using standard tools like `kubectl` and GitOps pipelines.
- **Design Rule**: Separate the user input (`spec`) from the operator's observations (`status`). Always enable the `/status` subresource.

---

## 3. Use Appropriate Kubernetes Abstractions Whenever Possible
Do not reinvent the wheel. If standard Kubernetes components can achieve the desired state, wrap them as child resources.
- **Rationale**: Utilizing built-in types (like `StatefulSet`, `Deployment`, `Job`, `Service`, `ConfigMap`) ensures you inherit years of robust scheduling, replication, and networking logic built into the core Kubernetes controllers.
- **Bad Practice**: Reconciling raw containers, writing custom pod-scheduler code inside the Operator, or writing custom network routing loops.
- **Good Practice**:
  - Use `StatefulSet` to run databases and stateful applications that require stable network IDs and persistent storage.
  - Use `Deployment` for stateless web servers or API layers.
  - Use `Job` or `CronJob` to handle migrations, backups, and database initialization tasks.

---

## 4. Operator Termination Must Not Affect the Operand
Deleting or stopping the Operator deployment must **never** disrupt or crash the managed application (the operand).
- **Rationale**: An Operator is the administrative control plane, not the data plane. If the Operator crashes or is being upgraded, the running database clusters, web servers, or caches must continue to serve traffic.
- **Design Rules**:
  - The managed pods must run independently of the Operator's runtime status.
  - Avoid design patterns where child resources periodically ping the Operator for health checks or configuration syncs.
  - Ensure that resource cleanups are managed via garbage collection owner references, but keep the data plane decoupled.

---

## 5. Understand Resource Types Created by Any Previous Versions
An Operator must be backward compatible. It must be able to parse, upgrade, and reconcile custom resources created by previous versions of the Operator.
- **Rationale**: Upgrading an Operator in a production cluster must be a zero-downtime operation. If version 2 cannot parse the schema of version 1, existing deployments will break.
- **Best Practices**:
  - Follow semantic versioning (`v1alpha1` -> `v1beta1` -> `v1`).
  - Implement **conversion webhooks** if the schema changes between versions.
  - Never remove old fields from the Go struct representation without establishing a deprecation cycle and automated migration path.

---

## 6. Coordinate Application Upgrades
Upgrading the managed application is a complex operation that must be orchestrated by the Operator.
- **Rationale**: Simply changing the container image in a StatefulSet template might trigger a blunt rolling restart that corrupts database quorums. The Operator must manage this process.
- **Upgrade Orchestration Rules**:
  - Validate the health of every node in the cluster before starting the upgrade.
  - Upgrade nodes sequentially or in safe quorums (e.g., upgrade database followers first, verify sync, failover the leader, then upgrade the former leader).
  - Automatically roll back to the previous stable version if health checks fail during the roll out.

---

## 7. Thoroughly Tested, Including Chaos Testing
Operators manage critical stateful systems. You must trust the Operator to handle cluster degradation.
- **Rationale**: Traditional unit tests do not capture real-world network partitions, disk space issues, or spontaneous node deaths.
- **Chaos Testing Checklist**:
  - **Pod Death**: Delete random child pods while the application is under active write load. The Operator must restore them without data loss.
  - **Network Partitions**: Simulate network cuts between the Operator and the API server, or between database replicas.
  - **Node Failures**: Terminate VM nodes hosting the managed cluster. Verify that failovers trigger correctly.
