# Quickstart for Kubernetes

This walkthrough demonstrates how to deploy the SPIRE Server and SPIRE Agent running in a Kubernetes cluster, and configuring a workload container to access SPIRE. The original tutorial can be found [here](https://spiffe.io/docs/latest/try/getting-started-k8s/).

The content supplements the existing Quickstart for Kubernetes guide with considerations for a deployment on OpenShift.

## Implementation

### Prerequisites

The following must be made available prior to starting the quickstart:

* Content from the SPIRE Tutorials repository on your machine and in particular, assets in the `k8s/quickstart` directory. This directory will be referred to as `$SPIRE_TUTORIALS`
* Content from this repository and in particular, assets in the `k8s-quickstart-openshift` directory. This directory will be referred to as `$SPIRE_OPENSHIFT`
* Cluster Administrator access to the OpenShift environment.

### Security Context Context Constraints (SCC)

OpenShift, by default, restricts various types of access that an application might request or expect during its typical operation. Given that SPIRE requires elevated access at a host level, a customized SCC must be created. Application can then be provided access to the newly created SCC. Additional details are outlined [here](https://github.com/spiffe/spire/blob/master/doc/spire_agent.md#openshift-support).

As outlined in the SPIRE Tutorial, create a new namespace called `spire`

```shell
$ kubectl apply -f $SPIRE_TUTORIALS/spire-namespace.yaml
```

Now create a SCC called `spire`:

```shell
$ kubectl apply -f $SPIRE_OPENSHIFT/spire-scc.yaml
```

Create a _ClusterRole_ called `system:openshift:scc:spire` to allow workloads to make use of the spire SCC


```shell
$ kubectl apply -f $SPIRE_OPENSHIFT/spire-scc-clusterrole.yaml
```

### Configure the SPIRE Server and Agent

No additional permissions are required by SPIRE server when running in OpenShift. The steps listed in the **Configure SPIRE Server** can be completed as documented.

Change into the `$SPIRE_TUTORIALS` directory and follow the directions in the **Configure SPIRE Server** section:

```shell
$ cd $SPIRE_TUTORIALS
```

Then complete the tasks as described.

Once complete, return to the prior directory

```shell
$ popd
```

Since the SPIRE agent does require host level access to OpenShift nodes, which would be blocked by default when using the `restricted` SCC.

First, create the _ServiceAccount_ and _ClusterRole_ as documented:

```shell
$ kubectl apply \
    -f $SPIRE_TUTORIALS/agent-account.yaml \
    -f $SPIRE_TUTORIALS/agent-cluster-role.yaml
```

With the `spire-agent` _ServiceAccount_ created, create a _RoleBinding_ that will bind the _ServiceAccount_ to the `system:openshift:scc:spire` _ClusterRole_ created previously, enabling the workload access to the `spire` SCC.  

```shell
$ kubectl apply -f $SPIRE_OPENSHIFT/spire-agent-scc-rolebinding.yaml
```

Create the SPIRE agent ConfigMap and DaemonSet.

```shell
$ kubectl apply \
    -f $SPIRE_TUTORIALS/agent-configmap.yaml \
    -f $SPIRE_TUTORIALS/agent-daemonset.yaml
```

At this point, you should see a single SPIRE server pod and a SPIRE agent pod for each Kubernetes Node.

```shell
$ kubectl get pods -n spire

spire-agent-9d88z   1/1     Running   0          106s
spire-agent-m8svp   1/1     Running   0          106s
spire-agent-qxgch   1/1     Running   0          106s
spire-server-0      1/1     Running   0          3m48s
```

If no SPIRE agent pods, confirm that the steps to configure the SCC and policies granting the agent service account access were completed successfully. If an error was discovered or a step omitted, there may be a delay before the _DaemonSet_ rolls out the pods. You may need to delete and recreate the DaemonSet.

### Register Workloads

Follow the steps as described in the tutorial to create a new registration entry for the node:

```shell
$ kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s_sat:cluster:demo-cluster \
    -selector k8s_sat:agent_ns:spire \
    -selector k8s_sat:agent_sa:spire-agent \
    -node
```

The quickstart then attempts to create a new registration entry for the workload. The registration assumes that the target workload in located the `default` namespace using the `default` ServiceAccount. It is best practice to not deploy resources in the `default` namespace, but to also specify unique _ServiceAccount's_.

Let's create a new namespace called `spire-workload` to contain the sample workload:

```shell
$ kubectl apply -f $SPIRE_OPENSHIFT/spire-workload-namespace.yaml
```

Create a new ServiceAccount called `spire-workload` that will be used to run the workload

```shell
$ kubectl apply -f $SPIRE_OPENSHIFT/spire-workload-serviceaccount.yaml
```

Create a new Registration entry for the workload that references the newly created _Namespace_ and _ServiceAccount_

```shell
$ kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/spire-workload/sa/spire-workload \
    -parentID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s:ns:spire-workload \
    -selector k8s:sa:spire-workload
```

### Configure a Workload Container to Access SPIRE

The next step is to deploy the sample workload to the `spire-workload` namespace. Similar to the `spire-agent` _DaemonSet_, the sample workload requires access to resources not allowed by OpenShift by default. Create a _RoleBinding_ to enable the `spire-workload` access to the `spire` SCC

```shell
$ kubectl apply -f $SPIRE_OPENSHIFT/spire-workload-scc-rolebinding.yaml
```

Now, deploy the sample workload

```shell
$ kubectl apply -n spire-workload -f $SPIRE_TUTORIALS/client-deployment.yaml
```

Update the deployment to make use of the `spire-workload` _ServiceAccount_ previously created

```shell
$ kubectl set serviceaccount -n spire-workload deployment client spire-workload
```

Confirm the sample workload is running in the `spire-workload` namespace

```shell
$ kubectl get pods -n spire-workload

NAME                     READY   STATUS    RESTARTS   AGE
client-8f546d7fc-wz62b   1/1     Running   0          11s
```

Start a shell connection to the running pod:

```shell
$ kubectl exec -n spire-workload -it $(kubectl get pods -n spire-workload -o=jsonpath='{.items[0].metadata.name}' \
   -l app=client)  -- /bin/sh
```

Verify the container can access the socket

```shell
$ /opt/spire/bin/spire-agent api fetch -socketPath /run/spire/sockets/agent.sock

Received 1 svid after 13.448904ms

SPIFFE ID:              spiffe://example.org/ns/spire-workload/sa/spire-workload
SVID Valid After:       2021-07-20 16:38:36 +0000 UTC
SVID Valid Until:       2021-07-20 17:38:46 +0000 UTC
CA #1 Valid After:      2021-07-20 15:56:49 +0000 UTC
CA #1 Valid Until:      2021-07-21 15:56:59 +0000 UTC
```

At this point, you have completed the quickstart walkthrough!

### Tear Down All Components

1. Delete Namespaces

```shell
$ kubectl delete namespace spire-workload spire
```

2. Delete ClusterRoles and ClusterRoleBindings

```shell
$ kubectl delete clusterrole spire-server-trust-role spire-agent-cluster-role system:openshift:scc:spire
$ kubectl delete clusterrolebinding spire-server-trust-role-binding spire-agent-cluster-role-binding

```


