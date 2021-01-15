# acm-demo-journey
Basic walkthrough of Anthos Config Management capabilities ([Config Sync](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/overview) &amp; [Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller)) as well as integration with [Config Connector](https://cloud.google.com/config-connector/docs/overview)'s [KRM](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/resource-management.md) model for GCP resources.

This walkthrough assumes you have Anthos Config Management already installed (the operator, specifically) on your GKE cluster(s). If you haven't already done this, follow [these](https://cloud.google.com/anthos-config-management/docs/how-to/installing) instructions. If, on the other hand, you have no idea what Anthos Config Management is and you landed here by mistake, take a moment to familiarize yourself with ACM  via an overview [here](https://cloud.google.com/anthos-config-management/docs/overview).

This isn't meant to be a deep dive in to all the various bells and whistles that ACM provides - we're not going to touch things like [Binary Authorization](https://cloud.google.com/binary-authorization/docs/overview) nor the [Heirarchy Controller](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/concepts/hierarchy-controller) - but this should be enough to start building some muscle memory on how to simplify multi-cluster management *and* defining guardrails for configuration within those clusters. 

#### Setup

As mentioned in the prior section, we're going to assume you have the `config-management-operator` deployed to your GKE cluster as decscribed [here](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync#configuring-config-sync).

For the configuration of ACM beyond that, the following section includes a sample `configManagement` spec YAML that will get you up and running with minimal modification. 


##### 01-config-sync

Let's actually deploy the config sync, policy controller, and config connector bits first. There is a sample `config-management.yaml` in `01-config-sync/sample-config-management`. Tweak the cluster name if you wish, and apply it:

```
$ kubectl apply -f 01-config-sync/sample-config-management/config-management.yaml
configmanagement.configmanagement.gke.io/config-management created
```

A few things will happen after you apply that YAML. The pods for Config Sync, Policy Controller, and Config Connector will get deployed, and with Config Sync deployed, your cluster will now start watching the `policyDir` defined in `01-config-sync/sample-config-management/config-management.yaml`. If Config Sync is working properly, you'll have 3 additional namespaces (`audit`, `inherited-demo-01`, and `inherited-demo-01`):

```
$ kubectl get ns
configmanagement.configmanagement.gke.io/config-management created
NAME                       STATUS   AGE
audit                      Active   7m46s
********  OTHER NAMESPACE STUFF  ********
inherited-demo-01          Active   7m46s
inherited-demo-02          Active   7m46s
```

Verify that Config Sync is constantly reconciling the configuration of your cluster against the desired state in the `policyDir` by deleting the `audit` namespace, and verifying that it gets re-created automatically in near-realtime:

```
$ kubectl delete ns audit
namespace "audit" deleted

$ kubectl get ns audit
NAME    STATUS   AGE
audit   Active   28s
```

Last thing we're going to verify is that namespace inheritence is working. Namespace inheritence is described [here](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/concepts/namespace-inheritance). For our example, `01-config-sync/namespaces/inherited-example` is an "abstract namespace directory" that contains `inherited-configmap.yaml`, a sample `configMap`. Namespace inheritence allows us to define that namespace spec once, and the two namespaces defined (`inherited-demo-01` and `inherited-demo-02`) will both "inherit" the configMap.

```
$ kubectl get cm/inherited-configmap -oyaml -n inherited-demo-01
apiVersion: v1
data:
  database: foo-db
  database_uri: foo-db://1.1.1.1:1234
kind: ConfigMap

$ kubectl get cm/inherited-configmap -oyaml -n inherited-demo-02
apiVersion: v1
data:
  database: foo-db
  database_uri: foo-db://1.1.1.1:1234
kind: ConfigMap
```

