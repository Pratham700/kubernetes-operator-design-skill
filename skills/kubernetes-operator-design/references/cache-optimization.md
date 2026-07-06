# Informer Cache & OOMKill Prevention Guide

In Kubernetes operators built with `controller-runtime`, memory exhaustion (OOMKill) is a systemic risk due to unfiltered informer caching. This guide outlines how to prevent OOMKills, optimize informer caches, and avoid the five common caching anti-patterns.

---

## 1. Preventing OOMKills: Label-Filtered Caching
By default, watching a resource type registers a cluster-wide informer that stores *every* object of that type in the operator's memory cache. If an attacker floods the cluster with huge or numerous resources (like dummy ConfigMaps or Secrets), the operator will run out of memory.

### Remediation:
Apply a label selector filter to informer cache options so the manager only caches resources matching the operator's label.

```go
package main

import (
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/labels"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/cache"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

func main() {
	options := ctrl.Options{
		Cache: cache.Options{
			// Restrict caching to resources created/managed by this operator
			ByObject: map[client.Object]cache.ByObject{
				&corev1.ConfigMap{}: {
					Label: labels.SelectorFromSet(labels.Set{
						"app.kubernetes.io/managed-by": "my-operator",
					}),
				},
				&corev1.Secret{}: {
					Label: labels.SelectorFromSet(labels.Set{
						"app.kubernetes.io/managed-by": "my-operator",
					}),
				},
			},
		},
	}
	
	// Create manager with filtered cache options...
}
```

> [!IMPORTANT]
> **Upgrade Migration**: If you add label-filtering to an existing operator, the operator will "lose sight" of pre-existing managed resources that do not have the label. You must apply a **merge patch** to label all pre-existing managed resources *prior to* or *during* the operator's upgrade deployment.

---

## 2. Advanced Cache Memory Optimizations

### A. Managed Fields Stripping
Kubernetes 1.20+ includes metadata tracking via `managedFields`, which can take up to 90% of a resource's memory footprint. Strip this metadata from cached objects:

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
	Cache: cache.Options{
		// Strip managed fields globally before storing objects in cache
		DefaultTransform: cache.TransformStripManagedFields(),
	},
})
```

### B. Metadata-Only Caching
If your controller only needs to watch a resource to trigger reconciliation on changes (e.g. Pod updates) and does not read the resource's spec or status, configure the controller to watch **Metadata Only**:

```go
import (
	corev1 "k8s.io/api/core/v1"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/handler"
)

func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&myv1.MyCRD{}).
		// Watch Pods but only cache their ObjectMeta (reduces pod size in cache from ~500KB to ~1KB)
		WatchesMetadata(&corev1.Pod{}, &handler.EnqueueRequestForOwner{
			OwnerType:    &myv1.MyCRD{},
			IsController: true,
		}).
		Complete(r)
}
```

---

## 3. Five Caching Anti-Patterns & How to Avoid Them

### Anti-Pattern 1: "My predicate filters it, so I'm safe"
- **The Pitfall**: Assuming event predicates (like `WithEventFilter()`) prevent resources from being cached.
- **The Reality**: Predicates act *after* cache ingestion. They block events from triggering the reconciliation work queue, but the informer cache still downloads and stores the resource.
- **The Fix**: Filter the cache using `cache.Options.ByObject` options.

### Anti-Pattern 2: "I used DisableFor, so caching is off"
- **The Pitfall**: Thinking that setting `DisableFor` in client options disables informer watches.
- **The Reality**: `DisableFor` only tells the client to read directly from the API server for `Get()` and `List()`. If you still watch the resource type in `SetupWithManager` (via `Watches()`, `Owns()`, `For()`), the informer cache will still download and cache it, resulting in both high memory and high API server queries (double overhead!).
- **The Fix**: Remove the watched type from the builder if caching is disabled.

### Anti-Pattern 3: The Invisible Informer from client.Get()
- **The Pitfall**: Running `client.Get()` on an uncached resource type under the assumption it goes directly to the API server without side effects.
- **The Reality**: The default cached client will **implicitly spin up a new, cluster-wide informer** for that type at runtime, creating a "phantom watch" that consumes memory in the background.
- **The Fix**: 
  1. Set `ReaderFailOnMissingInformer: true` in manager options during development to fail immediately on uncached reads.
  2. Use `mgr.GetAPIReader()` inside reconcilers to execute direct, uncached queries to the API server:
     ```go
     // In your Reconcile loop:
     err := r.APIReader.Get(ctx, req.NamespacedName, &myUncachedResource)
     ```

### Anti-Pattern 4: Everything is Cluster-Wide by Default
- **The Pitfall**: Deploying operators to run cluster-wide out of convenience.
- **The Reality**: This causes massive memory overhead (caching all namespaces) and inflates RBAC permissions (requires broad ClusterRoles).
- **The Fix**: Scope the informer cache to target namespaces using `DefaultNamespaces` in `cache.Options`:
  ```go
  options := ctrl.Options{
      Cache: cache.Options{
          DefaultNamespaces: map[string]cache.Config{
              "my-managed-namespace": {},
          },
      },
  }
  ```

### Anti-Pattern 5: The Typed/Unstructured Cache Trap
- **The Pitfall**: Watching typed objects (like `&corev1.ConfigMap{}`) but calling unstructured clients (like `&unstructured.Unstructured{}`) to read them in the reconciler.
- **The Reality**: `controller-runtime` separates caches for Typed, Unstructured, and Metadata. If you mix representations, it will create two separate informer caches for the exact same resource type, doubling memory overhead.
- **The Fix**: Keep representations consistent: watch typed, read typed; or watch unstructured, read unstructured.
