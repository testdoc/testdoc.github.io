---
---

This example demonstrates how [resource quota](/docs/admin/admission-controllers/#resourcequota) and
[limitsranger](/docs/admin/admission-controllers/#limitranger) can be applied to a Kubernetes namespace.
See [ResourceQuota design doc](https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/docs/design/admission_control_resource_quota.md) for more information.

This example assumes you have a functional Kubernetes setup.

## Step 1: Create a namespace

This example will work in a custom namespace to demonstrate the concepts involved.

Let's create a new namespace called quota-example:

```shell
$ kubectl create namespace quota-example
namespace "quota-example" created
```

Note that `kubectl` commands will print the type and name of the resource created or mutated, which can then be used in subsequent commands: 

```shell
$ kubectl get namespaces
NAME            STATUS    AGE
default         Active    50m
quota-example   Active    2s
```

## Step 2: Apply a quota to the namespace

By default, a pod will run with unbounded CPU and memory requests/limits.  This means that any pod in the
system will be able to consume as much CPU and memory on the node that executes the pod.

Users may want to restrict how much of the cluster resources a given namespace may consume
across all of its pods in order to manage cluster usage.  To do this, a user applies a quota to
a namespace.  A quota lets the user set hard limits on the total amount of node resources (cpu, memory)
and API resources (pods, services, etc.) that a namespace may consume. In term of resources, Kubernetes
checks the total resource *requests*, not resource *limits* of all containers/pods in the namespace.

Let's create a simple quota in our namespace:

```shell
$ kubectl create -f docs/admin/resourcequota/quota.yaml --namespace=quota-example
resourcequota "quota" created
```

Once your quota is applied to a namespace, the system will restrict any creation of content
in the namespace until the quota usage has been calculated.  This should happen quickly.

You can describe your current quota usage to see what resources are being consumed in your
namespace.

```shell
$ kubectl describe quota quota --namespace=quota-example
Name:			quota
Namespace:		quota-example
Resource		Used	Hard
--------		----	----
cpu			0	20
memory			0	1Gi
persistentvolumeclaims	0	10
pods			0	10
replicationcontrollers	0	20
resourcequotas		1	1
secrets			1	10
services		0	5
```

## Step 3: Applying default resource requests and limits

Pod authors rarely specify resource requests and limits for their pods.

Since we applied a quota to our project, let's see what happens when an end-user creates a pod that has unbounded
cpu and memory by creating an nginx container.

To demonstrate, lets create a Deployment that runs nginx:

```shell
$ kubectl run nginx --image=nginx --replicas=1 --namespace=quota-example
deployment "nginx" created
```

This creates a Deployment "nginx" with its underlying resource, a ReplicaSet, which handles the creation and deletion of Pod replicas. Now let's look at the pods that were created.

```shell
$ kubectl get pods --namespace=quota-example
NAME      READY     STATUS    RESTARTS   AGE
```

What happened?  I have no pods!  Let's describe the ReplicaSet managed by the nginx Deployment to get a view of what is happening.
Note that `kubectl describe rs` works only on kubernetes cluster >= v1.2. If you are running older versions, use `kubectl describe rc` instead.
If you want to obtain the old behavior, use `--generator=run/v1` to create replication controllers. See [`kubectl run`](/docs/user-guide/kubectl/kubectl_run/) for more details. 

```shell
$ kubectl describe rs -l run=nginx --namespace=quota-example
Name:		nginx-2040093540
Namespace:	quota-example
Image(s):	nginx
Selector:	pod-template-hash=2040093540,run=nginx
Labels:		pod-template-hash=2040093540,run=nginx
Replicas:	0 current / 1 desired
Pods Status:	0 Running / 0 Waiting / 0 Succeeded / 0 Failed
No volumes.
Events:
  FirstSeen	LastSeen	Count	From				SubobjectPath	Type		Reason		Message
  ---------	--------	-----	----				-------------	--------	------		-------
  48s		26s		4	{replicaset-controller }			Warning		FailedCreate	Error creating: pods "nginx-2040093540-" is forbidden: Failed quota: quota: must specify cpu,memory
```

The Kubernetes API server is rejecting the ReplicaSet requests to create a pod because our pods
do not specify any memory usage *request*.

So let's set some default values for the amount of cpu and memory a pod can consume:

```shell
$ kubectl create -f docs/admin/resourcequota/limits.yaml --namespace=quota-example
limitrange "limits" created
$ kubectl describe limits limits --namespace=quota-example
Name:		limits
Namespace:	quota-example
Type		Resource	Min	Max	Default Request	Default Limit	Max Limit/Request Ratio
----		--------	---	---	---------------	-------------	-----------------------
Container	cpu		-	-	100m		200m		-
Container	memory		-	-	256Mi		512Mi		-
```

Now any time a pod is created in this namespace, if it has not specified any resource request/limit, the default
amount of cpu and memory per container will be applied, and the request will be used as part of admission control.

Now that we have applied default resource *request* for our namespace, our Deployment should be able to
create its pods.

```shell
$ kubectl get pods --namespace=quota-example
NAME                     READY     STATUS    RESTARTS   AGE
nginx-2040093540-miohp   1/1       Running   0          5s
```

And if we print out our quota usage in the namespace:

```shell
$ kubectl describe quota quota --namespace=quota-example
Name:			quota
Namespace:		quota-example
Resource		Used	Hard
--------		----	----
cpu			100m	20
memory			256Mi	1Gi
persistentvolumeclaims	0	10
pods			1	10
replicationcontrollers	1	20
resourcequotas		1	1
secrets			1	10
services		0	5
```

You can now see the pod that was created is consuming explicit amounts of resources (specified by resource *request*), and the usage is being tracked by the Kubernetes system properly.

## Summary

Actions that consume node resources for cpu and memory can be subject to hard quota limits defined by the namespace quota. The resource consumption is measured by resource *request* in pod specification.

Any action that consumes those resources can be tweaked, or can pick up namespace level defaults to meet your end goal.
