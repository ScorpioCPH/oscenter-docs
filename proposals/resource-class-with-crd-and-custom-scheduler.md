
## Resource Class with CRD and Custom Scheduler

### Use Cases

- I have different types of CPUs (e.g. Intel E3/E5) or GPUs (e.g. Nvidia K80/1080Ti) on homogenous nodes, and i want to schedule pods on these different nodes.
- I want some nodes to be dedicated for special workflow.

### Design Proposal

#### CRD

We use CRD to represent resource class with a resource class controller.
There are 3 components in this design:

- resource-class controller
- resource-class scheduler
- device plugin

Device plugin will provide hardware device level implementation and it is supported in v1.8 release. We need to implement controller and scheduler which should work well with device plugin.

#### resource-class controller

This controller is for watching `ResourceClass` as defined with a CustomResourceDefinition (CRD). It will implement basic operations of `ResourceClass` such as:

- Register a new custom resource of type `ResourceClass` using a CustomResourceDefinition (CRD), support both namespace and cluter scopes.
- Create/Delete/Get/List/Update instances of this new resource type `ResourceClass`.
- Attach labels on the nodes for this `ResourceClass`.
- Mark taints on nodes to declare these nodes are dedicated if needed.
- Handle `matchExpressions` for this `ResourceClass` (TBD).
- Band `ResourceClass` with `ResourceName` and `Devices` which are advertised by device plugin (TBD).

**Data Specification**

```go
// ResourceClass is a specification for a ResourceClass resource
type ResourceClass struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    // Required. Spec defines resources
    Spec ResourceClassSpec `json:"spec"`
    // Required.
    Status ResourceClassStatus `json:"status"`
}

// ResourceClassSpec is the spec for a ResourceClass resource
type ResourceClassSpec struct {
    // Required. ResourceName is advertised by device plugin.
    ResourceName string `json:"resourceName"`
    // Required. DevicesIDs is advertised by device plugin, we cached here.
    DevicesIDs []string `json:"deviceIDs"`
    // Required. A list of resource selector requirements.
    // The requirements are ANDed.
    MatchExpressions []ResourceSelectorRequirement
    // Required. An array of available nodes for this resource class.
    // If empty then this resource can't be scheduled.
    Nodes []NodeSpec `json:"nodes"`
}

// A resource selector requirement is a selector that contains values, a key, and an operator
// that relates the key and values.
type NodeSpec struct {
    // Name is the name of this node.
    // We will add label to this node with "ResourceClassName=true".
    Name string `json:"name"`
    // Dedicated represents whether this node is dedicated.
    // If true, we will add taint to this node with "ResourceClassName=true:NoSchedule".
    Dedicated bool `json:"dedicated"`
}

// A resource selector requirement is a selector that contains values, a key, and an operator
// that relates the key and values.
type ResourceSelectorRequirement struct {
    // The label key that the selector applies to.
    Key string
    // Represents a key's relationship to a set of values.
    // Valid operators are In, NotIn, Exists, DoesNotExist. Gt, and Lt.
    Operator ResourceSelectorOperator
    // An array of string values. If the operator is In or NotIn,
    // the values array must be non-empty. If the operator is Exists or DoesNotExist,
    // the values array must be empty. If the operator is Gt or Lt, the values
    // array must have a single element, which will be interpreted as an integer.
    // This array is replaced during a strategic merge patch.
    // +optional
    Values []string
}

// A resource selector operator is the set of operators that can be used in
// a resource selector requirement.
type ResourceSelectorOperator string

const (
    ResourceSelectorOpIn           ResourceSelectorOperator = "In"
    ResourceSelectorOpNotIn        ResourceSelectorOperator = "NotIn"
    ResourceSelectorOpExists       ResourceSelectorOperator = "Exists"
    ResourceSelectorOpDoesNotExist ResourceSelectorOperator = "DoesNotExist"
    ResourceSelectorOpGt           ResourceSelectorOperator = "Gt"
    ResourceSelectorOpLt           ResourceSelectorOperator = "Lt"
)

// ResourceClassStatus is the status for a ResourceClass resource
type ResourceClassStatus struct {
    // Required. default is true, If false then this resource can't be required.
    Enable bool `json:"enable"`
}

```

#### resource-class scheduler

This scheduler is a custom scheduler which should work well with default scheduler (kube-scheduler). Any pod which required `ResourceClass` will be scheduled using this scheduler.

This scheduler will go through all `ResourceClass` via `List` interface of resource-class controller. Filter these resources to get available nodes witch match `MatchExpressions`, then bind this pod to this node (by node selector and taints/tolerations).

### User story

 The cluster admin knows what kind of `Resource` are present on the different nodes therefore he can setup the cluster to enable `ResourceClass`:

- create resource class with available nodes

```yaml
apiVersion: caicloud.io/v1alpha1
kind: ResourceClass
metadata:
  name: intel-cpu-e3
spec:
  resourceName: intel.com/cpu
  deviceIDs:
    - E3-1
    - E3-2
    - E3-3
  matchExpressions:
    - key: caicloud.io/resourceClass
      operator: In
      values:
        - intel-low-cpu
  nodes:
    - name: node-1
      dedicated: true
    - name: node-2
      dedicated: false
    - name: node-3
      dedicated: true
status:
  enable: true

```

- setup match expressions of this resource class? (TBD)
- deploy pod with custom scheduler and specific nodeSelector

`ResourceClass` can be requested by pod using `matchExpressions` which in the `nodeSelector` field in your pod spec.
Here is an example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-class-test
  labels:
    app: resource-class-test
spec:
  schedulerName: resource-scheduler
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    intel-cpu-e3: "true"
```
