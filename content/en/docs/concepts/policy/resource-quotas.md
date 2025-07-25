---
reviewers:
- derekwaynecarr
title: Resource Quotas
api_metadata:
- apiVersion: "v1"
  kind: "ResourceQuota"
content_type: concept
weight: 20
---

<!-- overview -->

When several users or teams share a cluster with a fixed number of nodes,
there is a concern that one team could use more than its fair share of resources.

_Resource quotas_ are a tool for administrators to address this concern.

A resource quota, defined by a ResourceQuota object, provides constraints that limit
aggregate resource consumption per {{< glossary_tooltip text="namespace" term_id="namespace" >}}. A ResourceQuota can also
limit the [quantity of objects that can be created in a namespace](#object-count-quota) by API kind, as well as the total
amount of {{< glossary_tooltip text="infrastructure resources" term_id="infrastructure-resource" >}} that may be consumed by
API objects found in that namespace.

{{< caution >}}
Neither contention nor changes to quota will affect already created resources.
{{< /caution >}}

<!-- body -->

## How Kubernetes ResourceQuotas work

ResourceQuotas work like this:

- Different teams work in different namespaces. This separation can be enforced with
  [RBAC](/docs/reference/access-authn-authz/rbac/) or any other [authorization](/docs/reference/access-authn-authz/authorization/)
  mechanism.

- A cluster administrator creates at least one ResourceQuota for each namespace.
  - To make sure the enforcement stays enforced, the cluster administrator should also restrict access to delete or update
    that ResourceQuota; for example, by defining a [ValidatingAdmissionPolicy](/docs/reference/access-authn-authz/validating-admission-policy/).

- Users create resources (pods, services, etc.) in the namespace, and the quota system
  tracks usage to ensure it does not exceed hard resource limits defined in a ResourceQuota.

  You can apply a [scope](#quota-scopes) to a ResourceQuota to limit where it applies,

- If creating or updating a resource violates a quota constraint, the control plane rejects that request with HTTP
  status code `403 Forbidden`. The error includes a message explaining the constraint that would have been violated.

- If quotas are enabled in a namespace for {{< glossary_tooltip text="resource" term_id="infrastructure-resource" >}}
  such as `cpu` and `memory`, users must specify requests or limits for those values when they define a Pod; otherwise,
  the quota system may reject pod creation.

  The resource quota [walkthrough](/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)
  shows an example of how to avoid this problem.

{{< note >}}
* You can define a [LimitRange](/docs/concepts/policy/limit-range/)
  to force defaults on pods that make no compute resource requirements (so that users don't have to remember to do that).
{{< /note >}}

You often do not create Pods directly; for example, you more usually create a [workload management](/docs/concepts/workloads/controllers/)
object such as a {{< glossary_tooltip term_id="deployment" >}}. If you create a Deployment that tries to use more
resources than are available, the creation of the Deployment (or other workload management object) **succeeds**, but
the Deployment may not be able to get all of the Pods it manages to exist. In that case you can check the status of
the Deployment, for example with `kubectl describe`, to see what has happened.

- For `cpu` and `memory` resources, ResourceQuotas enforce that **every**
  (new) pod in that namespace sets a limit for that resource.
  If you enforce a resource quota in a namespace for either `cpu` or `memory`,
  you and other clients, **must** specify either `requests` or `limits` for that resource,
  for every new Pod you submit. If you don't, the control plane may reject admission
  for that Pod.
- For other resources: ResourceQuota works and will ignore pods in the namespace without
  setting a limit or request for that resource. It means that you can create a new pod
  without limit/request for ephemeral storage if the resource quota limits the ephemeral
  storage of this namespace.

You can use a [LimitRange](/docs/concepts/policy/limit-range/) to automatically set
a default request for these resources.

The name of a ResourceQuota object must be a valid
[DNS subdomain name](/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).

Examples of policies that could be created using namespaces and quotas are:

- In a cluster with a capacity of 32 GiB RAM, and 16 cores, let team A use 20 GiB and 10 cores,
  let B use 10GiB and 4 cores, and hold 2GiB and 2 cores in reserve for future allocation.
- Limit the "testing" namespace to using 1 core and 1GiB RAM. Let the "production" namespace
  use any amount.

In the case where the total capacity of the cluster is less than the sum of the quotas of the namespaces,
there may be contention for resources. This is handled on a first-come-first-served basis.


## Enabling Resource Quota

ResourceQuota support is enabled by default for many Kubernetes distributions. It is
enabled when the {{< glossary_tooltip text="API server" term_id="kube-apiserver" >}}
`--enable-admission-plugins=` flag has `ResourceQuota` as
one of its arguments.

A resource quota is enforced in a particular namespace when there is a
ResourceQuota in that namespace.

## Compute Resource Quota

You can limit the total sum of
[compute resources](/docs/concepts/configuration/manage-resources-containers/)
that can be requested in a given namespace.

The following resource types are supported:

| Resource Name | Description |
| ------------- | ----------- |
| `limits.cpu` | Across all pods in a non-terminal state, the sum of CPU limits cannot exceed this value. |
| `limits.memory` | Across all pods in a non-terminal state, the sum of memory limits cannot exceed this value. |
| `requests.cpu` | Across all pods in a non-terminal state, the sum of CPU requests cannot exceed this value. |
| `requests.memory` | Across all pods in a non-terminal state, the sum of memory requests cannot exceed this value. |
| `hugepages-<size>` | Across all pods in a non-terminal state, the number of huge page requests of the specified size cannot exceed this value. |
| `cpu` | Same as `requests.cpu` |
| `memory` | Same as `requests.memory` |

### Resource Quota For Extended Resources

In addition to the resources mentioned above, in release 1.10, quota support for
[extended resources](/docs/concepts/configuration/manage-resources-containers/#extended-resources) is added.

As overcommit is not allowed for extended resources, it makes no sense to specify both `requests`
and `limits` for the same extended resource in a quota. So for extended resources, only quota items
with prefix `requests.` are allowed.

Take the GPU resource as an example, if the resource name is `nvidia.com/gpu`, and you want to
limit the total number of GPUs requested in a namespace to 4, you can define a quota as follows:

* `requests.nvidia.com/gpu: 4`

See [Viewing and Setting Quotas](#viewing-and-setting-quotas) for more details.

## Storage Resource Quota

You can limit the total sum of [storage resources](/docs/concepts/storage/persistent-volumes/)
that can be requested in a given namespace.

In addition, you can limit consumption of storage resources based on associated storage-class.

| Resource Name | Description |
| ------------- | ----------- |
| `requests.storage` | Across all persistent volume claims, the sum of storage requests cannot exceed this value. |
| `persistentvolumeclaims` | The total number of [PersistentVolumeClaims](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) that can exist in the namespace. |
| `<storage-class-name>.storageclass.storage.k8s.io/requests.storage` | Across all persistent volume claims associated with the `<storage-class-name>`, the sum of storage requests cannot exceed this value. |
| `<storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims` | Across all persistent volume claims associated with the `<storage-class-name>`, the total number of [persistent volume claims](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) that can exist in the namespace. |

For example, if you want to quota storage with `gold` StorageClass separate from
a `bronze` StorageClass, you can define a quota as follows:

* `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`
* `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`

In release 1.8, quota support for local ephemeral storage is added as an alpha feature:

| Resource Name | Description |
| ------------- | ----------- |
| `requests.ephemeral-storage` | Across all pods in the namespace, the sum of local ephemeral storage requests cannot exceed this value. |
| `limits.ephemeral-storage` | Across all pods in the namespace, the sum of local ephemeral storage limits cannot exceed this value. |
| `ephemeral-storage` | Same as `requests.ephemeral-storage`. |

{{< note >}}
When using a CRI container runtime, container logs will count against the ephemeral storage quota.
This can result in the unexpected eviction of pods that have exhausted their storage quotas.
Refer to [Logging Architecture](/docs/concepts/cluster-administration/logging/) for details.
{{< /note >}}

## Object Count Quota

You can set quota for *the total number of one particular resource kind* in the Kubernetes API,
using the following syntax:

* `count/<resource>.<group>` for resources from non-core groups
* `count/<resource>` for resources from the core group

Here is an example set of resources users may want to put under object count quota:

* `count/persistentvolumeclaims`
* `count/services`
* `count/secrets`
* `count/configmaps`
* `count/replicationcontrollers`
* `count/deployments.apps`
* `count/replicasets.apps`
* `count/statefulsets.apps`
* `count/jobs.batch`
* `count/cronjobs.batch`

If you define a quota this way, it applies to Kubernetes' APIs that are part of the API server, and
to any custom resources backed by a CustomResourceDefinition. If you use
[API aggregation](/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) to
add additional, custom APIs that are not defined as CustomResourceDefinitions, the core Kubernetes
control plane does not enforce quota for the aggregated API. The extension API server is expected to
provide quota enforcement if that's appropriate for the custom API.
For example, to create a quota on a `widgets` custom resource in the `example.com` API group, use `count/widgets.example.com`.

When using such a resource quota (nearly for all object kinds), an object is charged
against the quota if the object kind exists (is defined) in the control plane.
These types of quotas are useful to protect against exhaustion of storage resources. For example, you may
want to limit the number of Secrets in a server given their large size. Too many Secrets in a cluster can
actually prevent servers and controllers from starting. You can set a quota for Jobs to protect against
a poorly configured CronJob. CronJobs that create too many Jobs in a namespace can lead to a denial of service.

There is another syntax only to set the same type of quota for certain resources.
The following types are supported:

| Resource Name | Description |
| ------------- | ----------- |
| `configmaps` | The total number of ConfigMaps that can exist in the namespace. |
| `persistentvolumeclaims` | The total number of [PersistentVolumeClaims](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) that can exist in the namespace. |
| `pods` | The total number of Pods in a non-terminal state that can exist in the namespace. A pod is in a terminal state if `.status.phase in (Failed, Succeeded)` is true. |
| `replicationcontrollers` | The total number of ReplicationControllers that can exist in the namespace. |
| `resourcequotas` | The total number of ResourceQuotas that can exist in the namespace. |
| `services` | The total number of Services that can exist in the namespace. |
| `services.loadbalancers` | The total number of Services of type `LoadBalancer` that can exist in the namespace. |
| `services.nodeports` | The total number of `NodePorts` allocated to Services of type `NodePort` or `LoadBalancer` that can exist in the namespace. |
| `secrets` | The total number of Secrets that can exist in the namespace. |

For example, `pods` quota counts and enforces a maximum on the number of `pods`
created in a single namespace that are not terminal. You might want to set a `pods`
quota on a namespace to avoid the case where a user creates many small pods and
exhausts the cluster's supply of Pod IPs.

You can find more examples on [Viewing and Setting Quotas](#viewing-and-setting-quotas).

## Quota Scopes

Each quota can have an associated set of `scopes`. A quota will only measure usage for a resource if it matches
the intersection of enumerated scopes.

When a scope is added to the quota, it limits the number of resources it supports to those that pertain to the scope.
Resources specified on the quota outside of the allowed set results in a validation error.

| Scope | Description |
| ----- | ----------- |
| `Terminating` | Match pods where `.spec.activeDeadlineSeconds` >= `0` |
| `NotTerminating` | Match pods where `.spec.activeDeadlineSeconds` is `nil` |
| `BestEffort` | Match pods that have best effort quality of service. |
| `NotBestEffort` | Match pods that do not have best effort quality of service. |
| `PriorityClass` | Match pods that references the specified [priority class](/docs/concepts/scheduling-eviction/pod-priority-preemption). |
| `CrossNamespacePodAffinity` | Match pods that have cross-namespace pod [(anti)affinity terms](/docs/concepts/scheduling-eviction/assign-pod-node). |
| `VolumeAttributesClass` | Match persistentvolumeclaims that references the specified [volume attributes class](/docs/concepts/storage/volume-attributes-classes). |

The `BestEffort` scope restricts a quota to tracking the following resource:

* `pods`

The `Terminating`, `NotTerminating`, `NotBestEffort` and `PriorityClass`
scopes restrict a quota to tracking the following resources:

* `pods`
* `cpu`
* `memory`
* `requests.cpu`
* `requests.memory`
* `limits.cpu`
* `limits.memory`

Note that you cannot specify both the `Terminating` and the `NotTerminating`
scopes in the same quota, and you cannot specify both the `BestEffort` and
`NotBestEffort` scopes in the same quota either.

The `scopeSelector` supports the following values in the `operator` field:

* `In`
* `NotIn`
* `Exists`
* `DoesNotExist`

When using one of the following values as the `scopeName` when defining the
`scopeSelector`, the `operator` must be `Exists`.

* `Terminating`
* `NotTerminating`
* `BestEffort`
* `NotBestEffort`

If the `operator` is `In` or `NotIn`, the `values` field must have at least
one value. For example:

```yaml
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - middle
```

If the `operator` is `Exists` or `DoesNotExist`, the `values` field must *NOT* be
specified.

### Resource Quota Per PriorityClass

{{< feature-state for_k8s_version="v1.17" state="stable" >}}

Pods can be created at a specific [priority](/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority).
You can control a pod's consumption of system resources based on a pod's priority, by using the `scopeSelector`
field in the quota spec.

A quota is matched and consumed only if `scopeSelector` in the quota spec selects the pod.

When quota is scoped for priority class using `scopeSelector` field, quota object
is restricted to track only following resources:

* `pods`
* `cpu`
* `memory`
* `ephemeral-storage`
* `limits.cpu`
* `limits.memory`
* `limits.ephemeral-storage`
* `requests.cpu`
* `requests.memory`
* `requests.ephemeral-storage`

This example creates a quota object and matches it with pods at specific priorities. The example
works as follows:

- Pods in the cluster have one of the three priority classes, "low", "medium", "high".
- One quota object is created for each priority.

Save the following YAML to a file `quota.yaml`.

{{% code_sample file="policy/quota.yaml" %}}

Apply the YAML using `kubectl create`.

```shell
kubectl create -f ./quota.yaml
```

```
resourcequota/pods-high created
resourcequota/pods-medium created
resourcequota/pods-low created
```

Verify that `Used` quota is `0` using `kubectl describe quota`.

```shell
kubectl describe quota
```

```
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     1k
memory      0     200Gi
pods        0     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

Create a pod with priority "high". Save the following YAML to a
file `high-priority-pod.yaml`.

{{% code_sample file="policy/high-priority-pod.yaml" %}}

Apply it with `kubectl create`.

```shell
kubectl create -f ./high-priority-pod.yaml
```

Verify that "Used" stats for "high" priority quota, `pods-high`, has changed and that
the other two quotas are unchanged.

```shell
kubectl describe quota
```

```
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         500m  1k
memory      10Gi  200Gi
pods        1     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

### Cross-namespace Pod Affinity Quota

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

Operators can use `CrossNamespacePodAffinity` quota scope to limit which namespaces are allowed to
have pods with affinity terms that cross namespaces. Specifically, it controls which pods are allowed
to set `namespaces` or `namespaceSelector` fields in pod affinity terms.

Preventing users from using cross-namespace affinity terms might be desired since a pod
with anti-affinity constraints can block pods from all other namespaces
from getting scheduled in a failure domain.

Using this scope operators can prevent certain namespaces (`foo-ns` in the example below)
from having pods that use cross-namespace pod affinity by creating a resource quota object in
that namespace with `CrossNamespacePodAffinity` scope and hard limit of 0:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: disable-cross-namespace-affinity
  namespace: foo-ns
spec:
  hard:
    pods: "0"
  scopeSelector:
    matchExpressions:
    - scopeName: CrossNamespacePodAffinity
      operator: Exists
```

If operators want to disallow using `namespaces` and `namespaceSelector` by default, and
only allow it for specific namespaces, they could configure `CrossNamespacePodAffinity`
as a limited resource by setting the kube-apiserver flag `--admission-control-config-file`
to the path of the following configuration file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: apiserver.config.k8s.io/v1
    kind: ResourceQuotaConfiguration
    limitedResources:
    - resource: pods
      matchScopes:
      - scopeName: CrossNamespacePodAffinity
        operator: Exists
```

With the above configuration, pods can use `namespaces` and `namespaceSelector` in pod affinity only
if the namespace where they are created have a resource quota object with
`CrossNamespacePodAffinity` scope and a hard limit greater than or equal to the number of pods using those fields.

### Resource Quota Per VolumeAttributesClass

{{< feature-state feature_gate_name="VolumeAttributesClass" >}}

PersistentVolumeClaims can be created with a specific [volume attributes class](/docs/concepts/storage/volume-attributes-classes/), and might be modified after creation. You can control a PVC's consumption of storage resources based on the associated volume attributes classes, by using the `scopeSelector` field in the quota spec.

The PVC references the associated volume attributes class by the following fields:

* `spec.volumeAttributesClassName`
* `status.currentVolumeAttributesClassName`
* `status.modifyVolumeStatus.targetVolumeAttributesClassName`

A quota is matched and consumed only if `scopeSelector` in the quota spec selects the PVC.

When the quota is scoped for the volume attributes class using the `scopeSelector` field, the quota object is restricted to track only the following resources:

* `persistentvolumeclaims`
* `requests.storage`

This example creates a quota object and matches it with PVC at specific volume attributes classes. The example works as follows:

- PVCs in the cluster have at least one of the three volume attributes classes, "gold", "silver", "copper".
- One quota object is created for each volume attributes class.

Save the following YAML to a file `quota-vac.yaml`.

{{% code_sample file="policy/quota-vac.yaml" %}}

Apply the YAML using `kubectl create`.

```shell
kubectl create -f ./quota-vac.yaml
```

```
resourcequota/pvcs-gold created
resourcequota/pvcs-silver created
resourcequota/pvcs-copper created
```

Verify that `Used` quota is `0` using `kubectl describe quota`.

```shell
kubectl describe quota
```

```
Name:                   pvcs-gold
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     10Gi


Name:                   pvcs-silver
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     20Gi


Name:                   pvcs-copper
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     30Gi
```

Create a pvc with volume attributes class "gold". Save the following YAML to a file `gold-vac-pvc.yaml`.

{{% code_sample file="policy/gold-vac-pvc.yaml" %}}

Apply it with `kubectl create`.

```shell
kubectl create -f ./gold-vac-pvc.yaml
```

Verify that "Used" stats for "gold" volume attributes class quota, `pvcs-gold` has changed and that the other two quotas are unchanged.

```shell
kubectl describe quota
```

```
Name:                   pvcs-gold
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   10Gi


Name:                   pvcs-silver
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     20Gi


Name:                   pvcs-copper
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     30Gi
```

Once the PVC is bound, it is allowed to modify the desired volume attributes class. Let's change it to "silver" with kubectl patch.

```shell
kubectl patch pvc gold-vac-pvc --type='merge' -p '{"spec":{"volumeAttributesClassName":"silver"}}'
```

Verify that "Used" stats for "silver" volume attributes class quota, `pvcs-silver` has changed, `pvcs-copper` is unchanged, and `pvcs-gold` might be unchanged or released, which depends on the PVC's status.

```shell
kubectl describe quota
```

```
Name:                   pvcs-gold
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   10Gi


Name:                   pvcs-silver
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   20Gi


Name:                   pvcs-copper
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     30Gi
```

Let's change it to "copper" with kubectl patch. 

```shell
kubectl patch pvc gold-vac-pvc --type='merge' -p '{"spec":{"volumeAttributesClassName":"copper"}}'
```

Verify that "Used" stats for "copper" volume attributes class quota, `pvcs-copper` has changed, `pvcs-silver` and `pvcs-gold` might be unchanged or released, which depends on the PVC's status. 

```shell
kubectl describe quota
```

```
Name:                   pvcs-gold
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   10Gi


Name:                   pvcs-silver
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   20Gi


Name:                   pvcs-copper
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   30Gi
```

Print the manifest of the PVC using the following command:

```shell
kubectl get pvc gold-vac-pvc -o yaml
```

It might show the following output:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gold-vac-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: default
  volumeAttributesClassName: copper
status:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 2Gi
  currentVolumeAttributesClassName: gold
  phase: Bound
  modifyVolumeStatus:
    status: InProgress
    targetVolumeAttributesClassName: silver
  storageClassName: default
```

Wait a moment for the volume modification to complete, then verify the quota again.

```shell
kubectl describe quota
```

```
Name:                   pvcs-gold
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     10Gi


Name:                   pvcs-silver
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     10
requests.storage        0     20Gi


Name:                   pvcs-copper
Namespace:              default
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  1     10
requests.storage        2Gi   30Gi
```

## Requests compared to Limits {#requests-vs-limits}

When allocating compute resources, each container may specify a request and a limit value for either CPU or memory.
The quota can be configured to quota either value.

If the quota has a value specified for `requests.cpu` or `requests.memory`, then it requires that every incoming
container makes an explicit request for those resources. If the quota has a value specified for `limits.cpu` or `limits.memory`,
then it requires that every incoming container specifies an explicit limit for those resources.

## Viewing and Setting Quotas

kubectl supports creating, updating, and viewing quotas:

```shell
kubectl create namespace myspace
```

```shell
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
    requests.nvidia.com/gpu: 4
EOF
```

```shell
kubectl create -f ./compute-resources.yaml --namespace=myspace
```

```shell
cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    pods: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
```

```shell
kubectl create -f ./object-counts.yaml --namespace=myspace
```

```shell
kubectl get quota --namespace=myspace
```

```none
NAME                    AGE
compute-resources       30s
object-counts           32s
```

```shell
kubectl describe quota compute-resources --namespace=myspace
```

```none
Name:                    compute-resources
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4
```

```shell
kubectl describe quota object-counts --namespace=myspace
```

```none
Name:                   object-counts
Namespace:              myspace
Resource                Used    Hard
--------                ----    ----
configmaps              0       10
persistentvolumeclaims  0       4
pods                    0       4
replicationcontrollers  0       20
secrets                 1       10
services                0       10
services.loadbalancers  0       2
```

kubectl also supports object count quota for all standard namespaced resources
using the syntax `count/<resource>.<group>`:

```shell
kubectl create namespace myspace
```

```shell
kubectl create quota test --hard=count/deployments.apps=2,count/replicasets.apps=4,count/pods=3,count/secrets=4 --namespace=myspace
```

```shell
kubectl create deployment nginx --image=nginx --namespace=myspace --replicas=2
```

```shell
kubectl describe quota --namespace=myspace
```

```
Name:                         test
Namespace:                    myspace
Resource                      Used  Hard
--------                      ----  ----
count/deployments.apps        1     2
count/pods                    2     3
count/replicasets.apps        1     4
count/secrets                 1     4
```

## Quota and Cluster Capacity

ResourceQuotas are independent of the cluster capacity. They are
expressed in absolute units. So, if you add nodes to your cluster, this does *not*
automatically give each namespace the ability to consume more resources.

Sometimes more complex policies may be desired, such as:

- Proportionally divide total cluster resources among several teams.
- Allow each tenant to grow resource usage as needed, but have a generous
  limit to prevent accidental resource exhaustion.
- Detect demand from one namespace, add nodes, and increase quota.

Such policies could be implemented using `ResourceQuotas` as building blocks, by
writing a "controller" that watches the quota usage and adjusts the quota
hard limits of each namespace according to other signals.

Note that resource quota divides up aggregate cluster resources, but it creates no
restrictions around nodes: pods from several namespaces may run on the same node.

## Limit Priority Class consumption by default

It may be desired that pods at a particular priority, such as "cluster-services",
should be allowed in a namespace, if and only if, a matching quota object exists.

With this mechanism, operators are able to restrict usage of certain high
priority classes to a limited number of namespaces and not every namespace
will be able to consume these priority classes by default.

To enforce this, `kube-apiserver` flag `--admission-control-config-file` should be
used to pass path to the following configuration file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: apiserver.config.k8s.io/v1
    kind: ResourceQuotaConfiguration
    limitedResources:
    - resource: pods
      matchScopes:
      - scopeName: PriorityClass
        operator: In
        values: ["cluster-services"]
```

Then, create a resource quota object in the `kube-system` namespace:

{{% code_sample file="policy/priority-class-resourcequota.yaml" %}}

```shell
kubectl apply -f https://k8s.io/examples/policy/priority-class-resourcequota.yaml -n kube-system
```

```none
resourcequota/pods-cluster-services created
```

In this case, a pod creation will be allowed if:

1. the Pod's `priorityClassName` is not specified.
1. the Pod's `priorityClassName` is specified to a value other than `cluster-services`.
1. the Pod's `priorityClassName` is set to `cluster-services`, it is to be created
   in the `kube-system` namespace, and it has passed the resource quota check.

A Pod creation request is rejected if its `priorityClassName` is set to `cluster-services`
and it is to be created in a namespace other than `kube-system`.

## {{% heading "whatsnext" %}}

- See a [detailed example for how to use resource quota](/docs/tasks/administer-cluster/quota-api-object/).
- Read the ResourceQuota [API reference](/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/)
- Learn about [LimitRanges](/docs/concepts/policy/limit-range/)
- You can read the historical [ResourceQuota design document](https://git.k8s.io/design-proposals-archive/resource-management/admission_control_resource_quota.md)
  for more information.
- You can also read the [Quota support for priority class design document](https://git.k8s.io/design-proposals-archive/scheduling/pod-priority-resourcequota.md).
