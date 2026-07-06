# Go Reconciler & Finalizers Code Guide

This guide provides clean, production-ready Go code templates using the `controller-runtime` library for implementing the reconciliation loop, managing child resource lifecycles, and implementing finalizers.

---

## 1. Structure of the Reconciliation Loop
The reconciler must be designed to fetch the target custom resource, handle deletion, and invoke sub-reconcilers to manage children.

```go
package controllers

import (
	"context"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	dbv1 "github.com/example/operator/api/v1"
)

type DatabaseReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

const dbFinalizer = "database.example.com/finalizer"

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// 1. Fetch the primary Custom Resource instance
	db := &dbv1.Database{}
	err := r.Get(ctx, req.NamespacedName, db)
	if err != nil {
		if errors.IsNotFound(err) {
			logger.Info("Database resource not found. Ignoring since object must be deleted.")
			return ctrl.Result{}, nil
		}
		logger.Error(err, "Failed to get Database resource")
		return ctrl.Result{}, err
	}

	// 2. Handle Deletion and Finalizer logic
	isMarkedForDeletion := db.GetDeletionTimestamp() != nil
	if isMarkedForDeletion {
		if controllerutil.ContainsFinalizer(db, dbFinalizer) {
			// Run pre-deletion cleanup steps (e.g. clean up external database resources)
			if err := r.finalizeDatabase(ctx, db); err != nil {
				logger.Error(err, "Failed to run finalizer cleanup steps")
				return ctrl.Result{}, err
			}

			// Remove our finalizer from the list and update the resource
			controllerutil.RemoveFinalizer(db, dbFinalizer)
			err := r.Update(ctx, db)
			if err != nil {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, nil
	}

	// 3. Register finalizer if it doesn't exist yet
	if !controllerutil.ContainsFinalizer(db, dbFinalizer) {
		controllerutil.AddFinalizer(db, dbFinalizer)
		err = r.Update(ctx, db)
		if err != nil {
			return ctrl.Result{}, err
		}
	}

	// 4. Invoke child reconcilers (Idempotent execution)
	result, err := r.reconcileResources(ctx, db)
	if err != nil {
		return ctrl.Result{}, err
	}

	return result, nil
}
```

---

## 2. Idempotent Child Resource Creation & Ownership
When creating child resources (e.g., a database StateFulSet or ConfigMap), check for existence first. If it does not exist, create it. If it exists, update its spec if it differs from the desired spec.

```go
func (r *DatabaseReconciler) reconcileResources(ctx context.Context, db *dbv1.Database) (ctrl.Result, error) {
	// Derive the child name dynamically based on the parent
	deploymentName := db.Name + "-server"

	// Define the desired Deployment
	desiredDeploy := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      deploymentName,
			Namespace: db.Namespace,
			Labels: map[string]string{
				"app.kubernetes.io/managed-by": "database-operator",
				"database.example.com/parent":  db.Name,
			},
		},
		Data: map[string]string{
			"db-name": db.Spec.DatabaseName,
		},
	}

	// CRITICAL: Set Owner Reference for automatic garbage collection on parent deletion
	if err := controllerutil.SetControllerReference(db, desiredDeploy, r.Scheme); err != nil {
		return ctrl.Result{}, err
	}

	// Check if the ConfigMap already exists in the namespace
	existingConfig := &corev1.ConfigMap{}
	err := r.Get(ctx, client.ObjectKey{Name: deploymentName, Namespace: db.Namespace}, existingConfig)
	
	if err != nil {
		if errors.IsNotFound(err) {
			// Resource does not exist, create it
			err = r.Create(ctx, desiredDeploy)
			if err != nil {
				return ctrl.Result{}, err
			}
			return ctrl.Result{RequeueAfter: 2 * time.Second}, nil
		}
		return ctrl.Result{}, err
	}

	// If it exists, verify if an update is needed (Idempotency check)
	if existingConfig.Data["db-name"] != db.Spec.DatabaseName {
		existingConfig.Data["db-name"] = db.Spec.DatabaseName
		err = r.Update(ctx, existingConfig)
		if err != nil {
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{}, nil
}
```

---

## 3. Finalization Logic (Pre-Deletion Cleanups)
Finalizers allow you to block resource deletion until specific cleanup tasks (like exporting database dumps to S3, or deleting records from external APIs) are complete.

```go
func (r *DatabaseReconciler) finalizeDatabase(ctx context.Context, db *dbv1.Database) error {
	logger := log.FromContext(ctx)
	logger.Info("Executing finalizer cleanup steps for Database", "name", db.Name)

	// PLACEHOLDER: Put your actual cleanup logic here.
	// E.g., dbClient.DeleteDatabase(db.Name)
	
	return nil
}
```

---

## 4. Child Resource Naming Conventions
- **Uniqueness**: Child resource names must be unique within a namespace.
- **Derived and Reproducible**: Derive child names by appending a suffix to the parent resource name (e.g., `db.Name + "-service"`).
- **Collision Avoidance**: Never hardcode names (e.g. `const childName = "database-service"`). If the user creates multiple custom resources in the same namespace, a hardcoded child name will trigger naming collisions and block deployments.
