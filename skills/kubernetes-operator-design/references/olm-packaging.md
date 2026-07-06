# OLM Packaging & Metadata Specification

Operator Lifecycle Manager (OLM) manages the discovery, installation, and upgrades of Operators in a cluster. This guide specifies how to structure the OLM metadata bundle (CSV, package manifest, descriptors, and install modes) to make your Operator production-ready.

---

## 1. The ClusterServiceVersion (CSV)
The CSV is the core OLM metadata manifest describing the Operator. It is stored as a custom resource of kind `ClusterServiceVersion`.

### Key Components of a CSV:
- **Metadata**: Name, version, description, maintainers, provider, icons (Base64), maturity level (e.g. `alpha`, `beta`, `stable`).
- **Install Specification**: The deployment manifest and required ServiceAccounts, Roles, and RoleBindings.
- **CRD Descriptors**: Mapping of Custom Resource fields to visual components in Kubernetes Web Consoles (like OpenShift).

---

## 2. Using Spec & Status Descriptors
Descriptors provide metadata about the fields in your Custom Resource `.spec` and `.status` schemas. UIs use this metadata to render form components (e.g., text fields, toggle switches, status bars) for user interaction.

Descriptors are declared under the `spec.customresourcedefinitions.owned` section of the CSV.

### Descriptor Properties:
- `displayName`: User-friendly label for the field.
- `description`: Detailed explanation of what the field configures.
- `path`: The dot-delimited JSON path to the field (e.g., `size` or `resources.limits.cpu`).
- `x-descriptors`: A list of UI hints defining the control type.

### Common Spec Descriptors (`specDescriptors`):
| UI Element | Descriptor String |
| :--- | :--- |
| **Number Field** | `urn:alm:descriptor:com.tectonic.ui:number` |
| **Boolean Switch** | `urn:alm:descriptor:com.tectonic.ui:booleanSwitch` |
| **Password Input** | `urn:alm:descriptor:com.tectonic.ui:password` |
| **Image Pull Policy** | `urn:alm:descriptor:com.tectonic.ui:imagePullPolicy` |
| **Resource Limits** | `urn:alm:descriptor:com.tectonic.ui:resourceRequirements` |

### Common Status Descriptors (`statusDescriptors`):
| UI Element | Descriptor String |
| :--- | :--- |
| **Pod Count** | `urn:alm:descriptor:com.tectonic.ui:podCount` |
| **Pod Status Grid** | `urn:alm:descriptor:com.tectonic.ui:podStatuses` |
| **Phase / Condition** | `urn:alm:descriptor:io.kubernetes.phase` |
| **Status Message** | `urn:alm:descriptor:text` |

### Descriptor YAML Example
```yaml
spec:
  customresourcedefinitions:
    owned:
    - name: databases.database.example.com
      version: v1
      kind: Database
      displayName: Database Instance
      description: Deploys and manages a stateful database cluster.
      specDescriptors:
      - path: size
        displayName: Cluster Size
        description: The number of replica database pods to run.
        x-descriptors:
        - 'urn:alm:descriptor:com.tectonic.ui:podCount'
      - path: databaseName
        displayName: Database Name
        description: The name of the database instance to initialize.
        x-descriptors:
        - 'urn:alm:descriptor:com.tectonic.ui:text'
      statusDescriptors:
      - path: phase
        displayName: Deployment Phase
        description: The current lifecycle phase of the database cluster.
        x-descriptors:
        - 'urn:alm:descriptor:io.kubernetes.phase'
```

---

## 3. Install Modes Scoping
The CSV must explicitly declare which namespaces the Operator is configured to watch using `installModes`.

```yaml
spec:
  installModes:
  - type: OwnNamespace
    supported: true       # Watches only the namespace where the Operator pod is deployed
  - type: SingleNamespace
    supported: true       # Watches a single namespace defined in the OperatorGroup target
  - type: MultiNamespace
    supported: false      # Watches a list of multiple namespaces
  - type: AllNamespaces
    supported: true       # Watches the entire cluster (requires ClusterRoles)
```

---

## 4. Channels, Subscriptions, and Upgrades
OLM deploys and updates Operators via a **Subscription** to a release **Channel** (e.g. `stable`, `fast`, `candidate`).

### Versioning and Deprecation Strategy:
1. **Semantic Versioning**: Always tag Operator image releases and CSV versions using SemVer (e.g., `v1.0.0`, `v1.0.1`, `v1.1.0`).
2. **Upgrade Path (`replaces`)**: Instruct OLM on how to transition between releases by using the `replaces` field in the new version's CSV.
   ```yaml
   # Inside my-operator.v1.1.0.clusterserviceversion.yaml
   metadata:
     name: my-operator.v1.1.0
   spec:
     version: 1.1.0
     replaces: my-operator.v1.0.1  # Tells OLM to upgrade from 1.0.1 to 1.1.0
   ```
3. **Immutable Releases**: Never modify a CSV file that has already been pushed to a catalog and deployed. Release modifications must always occupy a new SemVer version.
