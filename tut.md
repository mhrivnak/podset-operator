# Go Operator - PodSet

## Prerequisites

kind, minikube or some cluster

`kubectl cluster-info`

```
Kubernetes control plane is running at https://127.0.0.1:41829
CoreDNS is running at https://127.0.0.1:41829/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'
```
## Initialize Project and Create API

First we will create a new project and use the Operator-SDK and
Kubebuilder to scaffold a minimal operator.

`mkdir podset-operator && cd podset-operator`

Initialize a new Go-based Operator SDK project for the PodSet Operator:

Note: Be sure to substitute your GitHub handle for mhrivnak :)

`operator-sdk init --domain=example.com --repo=github.com/mhrivnak/podset-operator`


## Create API for Custom Resource

Now that we have the skeleton for a project, we need to create our API
in the form of a Kubernetes Custom Resource Definition (CRD), as well as
a controller to interact with that CRD.

`operator-sdk create api --group=app --version=v1alpha1 --kind=PodSet --resource --controller`

We should now see the api, config, and controllers directories.

## Hello World!

As we implement the controller, we will iteratively add imports. For
brevity and convenience, add the final imports now and uncomment as they
are used.

Edit `controllers/podset_controller.go`

```go
import (
	"context"
	// "reflect"

	// corev1 "k8s.io/api/core/v1"
	// "k8s.io/apimachinery/pkg/api/errors"
	// metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	// "k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	// "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log" // TODO(you) This one gets removed
	// ctrllog "sigs.k8s.io/controller-runtime/pkg/log"
	// "sigs.k8s.io/controller-runtime/pkg/predicate"

	// TODO(you) Make sure this is your repo!
	appv1alpha1 "github.com/mhrivnak/podset-operator/api/v1alpha1"
)
```
Let’s now observe the default `controllers/podset_controller.go` file,
starting with `SetupWithManager`.

```go
// SetupWithManager sets up the controller with the Manager.
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appv1alpha1.PodSet{}).
		Complete(r)
}
```

For us, the key line is `For(&appv1alpha1.PodSet{})`, which causes the
reconcile loop to be run each time a PodSet is created, updated, or deleted.

Let's begin by logging "Hello World".

Change the `Reconcile` function to:

```go
func (r *PodSetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrllog.FromContext(ctx)
	log.Info("Hello World")
	return ctrl.Result{}, nil
}
```

Next, we need to generate the CRD for a PodSet. We'll cover this in a
moment. For now just run:

```
make manifests
```

Make sure the module requirements are installed.

```
go mod tidy
```

Now we are ready to run our most basic operator!

Install the CRD (allowing PodSets to be created later) onto the cluster:

```
make install
```

Now run your operator locally

```
make run
```

```
kubectl apply -f config/samples/app_v1alpha1_podset.yaml
```
You'll see it startup and then you'll see if we were successful in
the logs.

```
1.66662492348084e+09	INFO	Hello World	{"controller": "podset", "controllerGroup": "app.example.com", "controllerKind": "PodSet", "podSet": {"name":"podset-sample","namespace":"default"}, "namespace": "default", "name": "podset-sample", "reconcileID": "5ede544e-8244-461c-b738-9f803d60cc9b"}
```

Our first Operator has reconciled its first CR!

### Cleanup

You can stop a locally running Operator with `Ctrl+C`

If you delete the CR, all of the objects created by its reconcile (none
right now) are deleted. But we will just remove the CRD, which will
remove any CRs.

```
make uninstall
```

## Adding fields to the API

In Kubernetes, every functional object (with some exceptions, i.e.
ConfigMap) includes `spec` and `status`. Kubernetes functions by reconciling
desired state (Spec) with the actual cluster state. We then record what
is observed (Status).

Go-based Operators are able to generate and regenerate some of the
crucial files, so you don't have to! Let’s inspect one of the files we
**are** supposed to change, `api/v1alpha1/podset_types.go` which defines
the PodSet API for the auto-generation.

`PodSetSpec` represents the desired state, (input comes from PodSet CR).
Users will need to tell the Operator how many Pods we want, so lets add
`Replicas`.

```go
type PodSetSpec struct {
	// Replicas is the desired number of pods for the PodSet
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=10
	Replicas int32 `json:"replicas,omitempty"`
}
```

Notice the `+kubebuilder` comment markers found throughout the file.
Operator-SDK makes use of a tool called `controler-gen` (from the
[controller-tools](https://github.com/kubernetes-sigs/controller-tools)
project) for generating utility code and Kubernetes YAML. More
information on markers for config/code generation can be found
[here](https://book.kubebuilder.io/reference/markers.html).


Let's go ahead and add the Status fields that we will eventually use.

```go
// PodSetStatus defines the observed state of PodSet
type PodSetStatus struct {
	PodNames          []string `json:"podNames"`
	AvailableReplicas int32    `json:"availableReplicas"`
}
```

**Important: Every time you modify a `*_types.go` file, you will need to update the
generated files!**

Regenerate `zz_generated.deepcopy.go` with:

`make generate`

Regenerate object YAMLs (including the CRDs!):

`make manifests`

Thanks to our comment markers, observe that we now have a newly
generated CRD YAML that reflects the `spec.replicas` OpenAPI v3 schema
validation.

`cat config/crd/bases/app.example.com_podsets.yaml`

Next, lets use our the `Replicas` field in our Reconcile loop.
Modify the PodSet controller logic at `controllers/podset_controller.go`:


```go
func (r *PodSetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrllog.FromContext(ctx)

	// Fetch the PodSet instance
	instance := &appv1alpha1.PodSet{}
	err := r.Get(ctx, req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found; it could have been deleted after
			// the reconcile request was queued. Owned objects (in our case,
			// Pods) are automatically garbage collected, so there is nothing
			// for us to do. Return and don't requeue.
			return ctrl.Result{}, nil
		}
		// Error reading the object. By returning an error, the library will log
		// that error and requeue the resource with backoff logic.
		return ctrl.Result{}, err
	}
	log.Info(fmt.Sprintf("CR has specified %v replicas", instance.Spec.Replicas))
	return ctrl.Result{}, nil
}
```

Reminder: If you forgot to cleanup after "Hello World", stop the
controller with `CTRL+C` and uninstall the PodSet CRD with `make
uninstall`. (Hint: you'll have to remember how to do this next time.)

Reinstall the updated CRD, start the controller with `make run`, and in
another session, the CR with `kubectl apply -f config/samples/app_v1alpha1_podset.yaml`

## Reporting back to the user with Status

Our controller is now able to read values from the CR, now it is time to
report back using the `PodSet.Status.PodNames`
and`PodSet.Status.AvailableReplicas`  fields we created in the previous
step.

Let's again edit our Reconcile function:

```go
func (r *PodSetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrllog.FromContext(ctx)

	// Fetch the PodSet instance
	instance := &appv1alpha1.PodSet{}
	err := r.Get(ctx, req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found; it could have been deleted after
			// the reconcile request was queued. Owned objects (in our case,
			// Pods) are automatically garbage collected, so there is nothing
			// for us to do. Return and don't requeue.
			return ctrl.Result{}, nil
		}
		// Error reading the object. By returning an error, the library will log
		// that error and requeue the resource with backoff logic.
		return ctrl.Result{}, err
	}
	log.Info(fmt.Sprintf("CR has specified %v replicas", instance.Spec.Replicas))

	podList := &corev1.PodList{}
	listOps := &client.ListOptions{Namespace: instance.Namespace}
	if err = r.List(ctx, podList, listOps); err != nil {
		return ctrl.Result{}, err
	}
	// Find matching pods that are in phase pending or running
	var available []corev1.Pod
	for _, pod := range podList.Items {
		// skip pods that are being deleted
		if pod.ObjectMeta.DeletionTimestamp != nil {
			continue
		}
		if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
			available = append(available, pod)
		}
	}
	numAvailable := int32(len(available))

	// collect names of available pods
	availableNames := []string{}
	for _, pod := range available {
		availableNames = append(availableNames, pod.ObjectMeta.Name)
	}

	// Update the status only if it differs from the previous status. That helps
	// reduce load on the api-server.
	status := appv1alpha1.PodSetStatus{
		PodNames:          availableNames,
		AvailableReplicas: numAvailable,
	}
	if !reflect.DeepEqual(instance.Status, status) {
		instance.Status = status
		err = r.Status().Update(ctx, instance)
		if err != nil {
			log.Error(err, "Failed to update PodSet status")
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
```

Now we can restart the Operator, delete and recreate the CR.

`kubectl get podsets podset-sample -o yaml`

Will give us back (abbreviated):

```yaml
apiVersion: app.example.com/v1alpha1
kind: PodSet
metadata:
  name: podset-sample
  namespace: default
spec:
  replicas: 3
status:
  availableReplicas: 0
  podNames: []
```

Of course the status fields are empty now because we haven't actually
created any pods yet!

## Creating Pods

At last, we get to actually create the Pods!

First, we need to tell the manager that it will own Pods too.

```go
// SetupWithManager sets up the controller with the Manager.
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appv1alpha1.PodSet{}).
		Owns(&corev1.Pod{}).
		Complete(r)
}
```

Next, we create a helper function for creating a Pod.

```go
// newPodForCR returns a pod with the same name/namespace as the CR
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod
	return &corev1.Pod{
               ObjectMeta: metav1.ObjectMeta{
                       GenerateName: cr.Name + "-pod",
                       Namespace:    cr.Namespace,
               },
               Spec: corev1.PodSpec{
                       Containers: []corev1.Container{
                               {
                                       Name:    "fancy-alpine",
                                       Image:   "quay.io/amacdona/strauss",
                                       Command: []string{"sleep", "3600"},
                               },
                       },
               },
       }
}
```

Finally, we create pods if there aren't enough, and remove pods if there
are too many. The whole controller should now look like this:

```go
package controllers

import (
	"context"
	"fmt"
	"reflect"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	ctrllog "sigs.k8s.io/controller-runtime/pkg/log"

	appv1alpha1 "github.com/mhrivnak/podset-operator/api/v1alpha1"
)

// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=app.example.com,resources=podsets,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=app.example.com,resources=podsets/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=app.example.com,resources=podsets/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the PodSet object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.12.2/pkg/reconcile
func (r *PodSetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrllog.FromContext(ctx)

	// Fetch the PodSet instance
	instance := &appv1alpha1.PodSet{}
	err := r.Get(ctx, req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found; it could have been deleted after
			// the reconcile request was queued. Owned objects (in our case,
			// Pods) are automatically garbage collected, so there is nothing
			// for us to do. Return and don't requeue.
			return ctrl.Result{}, nil
		}
		// Error reading the object. By returning an error, the library will log
		// that error and requeue the resource with backoff logic.
		return ctrl.Result{}, err
	}
	log.Info(fmt.Sprintf("CR has specified %v replicas", instance.Spec.Replicas))

	podList := &corev1.PodList{}
	listOps := &client.ListOptions{Namespace: instance.Namespace}
	if err = r.List(ctx, podList, listOps); err != nil {
		return ctrl.Result{}, err
	}
	// Find matching pods that are in phase pending or running
	var available []corev1.Pod
	for _, pod := range podList.Items {
		// skip pods that are being deleted
		if pod.ObjectMeta.DeletionTimestamp != nil {
			continue
		}
		if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
			available = append(available, pod)
		}
	}
	numAvailable := int32(len(available))

	// collect names of available pods
	availableNames := []string{}
	for _, pod := range available {
		availableNames = append(availableNames, pod.ObjectMeta.Name)
	}

	// Update the status only if it differs from the previous status. That helps
	// reduce load on the api-server.
	status := appv1alpha1.PodSetStatus{
		PodNames:          availableNames,
		AvailableReplicas: numAvailable,
	}
	if !reflect.DeepEqual(instance.Status, status) {
		instance.Status = status
		err = r.Status().Update(ctx, instance)
		if err != nil {
			log.Error(err, "Failed to update PodSet status")
			return ctrl.Result{}, err
		}
	}

	if numAvailable < instance.Spec.Replicas {
		log.Info("Scaling up pods", "Currently available", numAvailable, "Required replicas", instance.Spec.Replicas)
		// Define a new Pod object
		pod := newPodForCR(instance)
		// Set PodSet instance as the owner and controller
		if err := controllerutil.SetControllerReference(instance, pod, r.Scheme); err != nil {
			return ctrl.Result{}, err
		}
		err = r.Create(ctx, pod)
		if err != nil {
			log.Error(err, "Failed to create pod", "pod.name", pod.Name)
			return ctrl.Result{}, err
		}

		if numAvailable > instance.Spec.Replicas {
			log.Info("Scaling down pods", "Currently available", numAvailable, "Required replicas", instance.Spec.Replicas)
			diff := numAvailable - instance.Spec.Replicas
			dpods := available[:diff]
			for i := range dpods {
				err = r.Delete(ctx, &dpods[i])
				if err != nil {
					log.Error(err, "Failed to delete pod", "pod.name", dpods[i].Name)
					return ctrl.Result{}, err
				}
			}
		}

	}

	log.Info("reconcile succeeded")
	return ctrl.Result{}, nil
}

// newPodForCR returns a pod with the same name/namespace as the cr
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			GenerateName: cr.Name + "-pod",
			Namespace:    cr.Namespace,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "fancy-alpine",
					Image:   "quay.io/amacdona/strauss",
					Command: []string{"sleep", "3600"},
				},
			},
		},
	}
}

// SetupWithManager sets up the controller with the Manager.
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appv1alpha1.PodSet{}).
		Owns(&corev1.Pod{}).
		Complete(r)
}
```

Once again, delete the CR, restart the controller, and recreate the CR.

Verify the PodSet exists:

`kubectl get podsets`

Verify the PodSet operator has created 3 pods:

`kubectl get pods`

Verify that status shows the name of the pods currently owned by the PodSet:

`kubectl get podset podset-sample -o yaml`

Increase the number of replicas owned by the PodSet:

`kubectl patch podset podset-sample --type='json' -p '[{"op": "replace", "path": "/spec/replicas", "value":5}]'`

(Alternatively `kubectl edit podset podset-sample` lets you change the
resource with your editor.)

Verify that we now have 5 running pods, and that the PodSet status matches
reality.

`kubectl get pods`
`kubectl get podset podset-sample -o yaml`

Lets see if it can scale down too.

`kubectl patch podset podset-sample --type='json' -p '[{"op": "replace", "path": "/spec/replicas", "value":1}]'`
`kubectl get pods`
`kubectl get podset podset-sample -o yaml`

Our PodSet controller creates pods containing OwnerReferences in their metadata section. This ensures they will be removed upon deletion of the podset-sample CR.

Observe the OwnerReference set on a PodSet’s pod:

`kubectl get pods -o yaml | grep ownerReferences -A10`

## Finishing up with Best Practices


### Do: Use labels when querying

Let's make sure that we are only looking for the Pods our own Operator
made, rather than **all** the Pods in the namespace. This is crucial for
correctness and performance, particularly on large clusters.

Add labelSelector to the listOpts.

```go
// List all pods owned by this PodSet instance
listOps := &client.ListOptions{Namespace: instance.Namespace}
lbs := map[string]string{
		"app":     instance.Name,
 		"version": "v0.1",
 	}
labelSelector := labels.SelectorFromSet(lbs)
listOps := &client.ListOptions{Namespace: instance.Namespace, LabelSelector: labelSelector}
```

You will also need to add the labels to the newPodForCR helper

```go
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
	labels := map[string]string{
		"app":     cr.Name,
		"version": "v0.1",
	}
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			GenerateName: cr.Name + "-pod",
			Namespace:    cr.Namespace,
			Labels:       labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "fancy-alpine",
					Image:   "quay.io/amacdona/strauss",
					Command: []string{"sleep", "3600"},
				},
			},
		},
	}
}
```

### Do Not: Reconcile on no-op updates

Predicates filter events, preventing the Reconcile from running when
unnecessary. Controller runtime provides predicates, or you can
implement your own.

```go
// SetupWithManager sets up the controller with the Manager.
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appv1alpha1.PodSet{}).
		Owns(&corev1.Pod{}).
		// This predicate will filter out update events that don't change the
		// resource's generation, such as status updates. Often a controller
		// doesn't need to reconcile when only the status of a resource changed,
		// and in those situation, this predicate reduces the number of times
		// Reconcile gets called.
		WithEventFilter(predicate.GenerationChangedPredicate{}).
		Complete(r)
}
```

### Do: Limit container permissions

```go
// newPodForCR returns a pod with the same name/namespace as the CR
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
	labels := map[string]string{
		"app":     cr.Name,
		"version": "v0.1",
	}
	yes := true
	no := false
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			GenerateName: cr.Name + "-pod",
			Namespace:    cr.Namespace,
			Labels:       labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "fancy-alpine",
					Image:   "quay.io/amacdona/strauss",
					Command: []string{"sleep", "3600"},
					// security best practices
					SecurityContext: &corev1.SecurityContext{
						AllowPrivilegeEscalation: &no,
						Capabilities: &corev1.Capabilities{
							Drop: []corev1.Capability{"ALL"},
						},
						RunAsNonRoot: &yes,
						SeccompProfile: &corev1.SeccompProfile{
							Type: corev1.SeccompProfileTypeRuntimeDefault,
						},
					},
				},
			},
		},
	}
}
```
