# DO380-Notes

### Moving from K8s to OCP

The `oc config get-contexts` command lists the contexts in the `kubeconfig` file.
~~~
$ oc config get-contexts
~~~

The `oc config use-context` command changes the current context.
~~~
$ oc config use-context default/api-ocp-example-com:6443/admin
~~~

The `oc config set-context` command updates a context.
~~~
$ oc config set-context /api-ocp-example-com:6443/developer --namespace=namespace
~~~

### Kubernetes Kustomize

Kustomize is a Kubernetes templating system that simplifies the deployment of your applications
on several environments or clusters.

The `kubectl` command integrates the `kustomization` tool. A kustomization is a directory
containing a `kustomization.yml`

~~~
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yml
~~~

The `resources` field is a list of files containing Kubernetes resources.
A `kustomization.yml` file can contain fields that describe modifications to be made to the
resources.
The `images` field describes modifications to images in the kustomization.
~~~
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
images:
  - name: image
    newName: new-image
    newTag: new-tag
~~~
The image *image* will be replaced by *new-image* with the tag *new-tag*.

`kustomization.yml` can contain a `bases` field. The `bases` field is a list of other kustomization
directories.
~~~
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - path-to-kustomization
~~~

Using kustomization
~~~
$ kubectl apply -k directory_name
~~~

### Extracting Information from Resources

The following example command demonstrates how to list resources of a specific type located
within a `namespace` using the command `oc get type -n namespace -o json`:
~~~
$ oc get deployment -n openshift-cluster-samples-operator -o json
~~~
**The following command demonstrates how to get information about the "schema" of an object or
its properties:**
~~~
$ oc explain deployment.status.replicas
KIND: Deployment
VERSION: apps/v1

FIELD:  replicas <integer>

DESCRIPTION:
     Total number of non-terminated pods targeted by this deployment (their
     labels match the selector).
~~~
 
**Extracting Information from a Single Resource**
~~~
$ oc get deployment cluster-samples-operator -n openshift-cluster-samples-operator -o jsonpath='{.status.availableReplicas}'

True
~~~

**Extracting a Single Property from Multiple Resources**
~~~
$ oc get route -n openshift-monitoring -o jsonpath='{.items[*].spec.host}'

alertmanager-main-openshift-monitoring.apps.ocp4.example.com
grafana-openshift-monitoring.apps.ocp4.example.com prometheus-k8s-
openshift-monitoring.apps.ocp4.example.com thanos-querier-openshift-
monitoring.apps.ocp4.example.com
~~~

**Extracting a Single Property with Multiple Nesting from Resources**

~~~
$ oc get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].image}'

quay.io/external_storage/nfs-client-provisioner:latest
...output omitted...
~~~

**Extracting Multiple Properties at Different Levels of Nesting from Resources**

~~~
$ oc get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace} {.metadata.creationTimestamp}{"\n"}{end}'

nfs-client-provisioner 2022-05-18T09:53:59Z
openshift-apiserver-operator 2022-05-18T09:55:10Z
openshift-apiserver 2022-05-18T10:54:46Z
openshift-apiserver 2022-05-18T10:53:25Z
...output omitted...
~~~

**Using Label Filtering**

~~~
$ oc get deployment -n openshift-cluster-storage-operator --show-labels

NAME
 ... LABELS
cluster-storage-operator
 ... <none>
csi-snapshot-controller
 ... <none>
csi-snapshot-controller-operator
 ... app=csi-snapshot-controller-operator
 ~~~

 ~~~
 $ oc get deployment -n openshift-cluster-storage-operator -l app=csi-snapshot-controller-operator -o name
 ~~~

 ## Deploying Scripts on OpenShift


# Storage

---

|OpenShift Storage Component| Description |
|---|---|
|Provisioner| Code that OpenShift uses to create a storage resource.|
|Persistent Volume|A cluster resource that defines the information that OpenShift requires to mount a volume to a node.|
|Storage Class|A cluster resource that defines characteristics for a particular type of storage that users can request.|
|Persistent Volume Claim|A project resource that defines a request for storage with specific storage characteristics.|
|Volume Plug-in | Code that OpenShift uses to mount a storage resource on a node.|



 Display all storage-related package manifests.
~~~
$ oc get packagemanifests | egrep 'NAME|storage'
~~~

Find the source values that you need for the subscription resource template.
~~~
$  oc describe packagemanifests local-storage-operator | grep -i 'Catalog Source'
~~~

Find the channel value that you need for the subscription resource template.
~~~
$  oc get packagemanifests local-storage-operator -o jsonpath='{range .status.channels[*]}{.name}{"\n"}{end}'
~~~

Display the list of custom resource definition types that the operator owns. Verify that no operator-owned custom resources exist in the project.
~~~
$ oc get clusterserviceversions -o name
clusterserviceversion.operators.coreos.com/local-storage-operator.4.10.0-202209080237

$ export CSV_NAME=$(oc get csv -o name)
$ echo ${CSV_NAME}
clusterserviceversion.operators.coreos.com/local-storage-operator.4.10.0-202209080237

$ oc get ${CSV_NAME} -o jsonpath='{.spec.customresourcedefinitions.owned[*].kind}{"\n"}'
LocalVolume LocalVolumeSet LocalVolumeDiscovery LocalVolumeDiscoveryResult

~~~

