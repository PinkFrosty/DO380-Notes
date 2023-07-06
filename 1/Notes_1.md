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