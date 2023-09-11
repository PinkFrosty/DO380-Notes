# DO380-Notes

Chapters

  - OpenShift CLI developer command reference - contexts
  - Images - Triggering updates on image stream changes
  - Authentication and authorization - rolebinding, LDAP
  - Nodes - Jobs
  - Operators - well....
  - Security and compliance - certs

# Authenticating to OpenShift

Kube path
~~~
~/.kube/config
~~~

The `oc login` command creates or updates `~/.kube/config`

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

*Tidbit* 
You can add a new image to the kustomization.yaml

~~~
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - path-to-kustomization
images:
  - name: registry.new.example.com:8443/versioned-hello
    newTag: v1.1
~~~

Using kustomization
~~~
$ kubectl apply -k directory_name
~~~
Use the `kubectl apply -k directory_name` command to apply a kustomization.

### Annotating Deployments with Image Stream Triggers
Enhance Kubernetes deployments with OpenShift image stream tags by adding the following metadata annotation.

~~~
{
  "image.openshift.io/triggers": "[
        {
           \"from\": {
               \"kind\":\"ImageStreamTag\",
               \"name\":\"versioned-hello:latest\"
            },
          \"fieldPath\":\"spec.template.spec.containers[?(@.name==\\\"hello\\
\")].image\"
        }
    ]"
}
~~~
You can retrieve that metadata annotation from a deployment by using the `oc get deploy/DEPLOYMENT_NAME -o yaml``.

*Tidbit*
Use `skopeo` to copy an image. 
~~~
$ skopeo copy docker://<registry.com/repo/image:v1.0> docker://<registry.com/repo/image:latest> 
~~~

Set the trigger annotation.
~~~
$ oc set trigger deploy/DEPLOYMENT_NAME --from-image IMAGE_STREAM_TAG -c CONTAINER_NAME.

$ oc set triggers deployment/hello --from-image example:latest -c hello
~~~

You can `get` for more than 1 resource.
~~~
oc get deployments,pods,services
~~~

# Introducing Automation with OpenShift
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

When using `oc exaplain deployment` or another resource you can keep going down by using `.`. I considered it the json path. `oc explain deployment.spec.selector`.

 
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

**Service Accounts**

~~~
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup_sa
  namespace: database
~~~

***Tid Bit***
Use the `oc` with `--dry-run=client` command to create a yaml output for you.
~~~
$ oc create sa newer -n test -o yaml --dry-run=client
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: newer
  namespace: test
~~~

**Defining Roles and Role Bindings**

A RoleBinding resource is namespaced. It associates accounts with roles within its namespace.

A ClusterRoleBinding resource is not namespaced and applies to all resources in the cluster.

~~~
$ oc describe clusterrole view

Name:  view
Labels:
   kubernetes.io/bootstrapping=rbac-defaults
   rbac.authorization.k8s.io/aggregate-to-edit=true
Annotations:vcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccczzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule: Resources Non-Resource URLs Resource Names  Verbs

---------  ----------------- --------------  -----
appliedclusterresourcequotas []  []  [get list watch]
bindings                     []  []  [get list watch]
buildconfigs/webhooks        []  []  [get list watch]
buildconfigs                 []  []  [get list watch]
...output omitted...
~~~

Clusterrolebinding
~~~
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backup
  namespace: database
subjects:
- kind: ServiceAccount
  name: backup_sa
  namespace: database
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-reader
~~~

Alternatively, use the oc policy add-role-to-user command to create or modify role bindings.

Create a custom role.
~~~
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configurator
  namespace: database
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create"]
~~~  

### Creating Jobs and Cron Jobs
A Job creates and executes pods until they successfully complete their task. Jobs can be configured to run a single pod or many pods concurrently. 

~~~
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
  namespace: database
spec:
  activeDeadlineSeconds: 600 (1)
  parallelism: 2 (2)
  template: (3)
    metadata:
      name: backup
    spec:
      serviceAccountName: backup_sa (4)
      containers:
      - name: backup
        image: example/backup_maker:v1.2.3
      restartPolicy: OnFailure
~~~
1 Optionally, provide a duration limit in seconds. The Job will attempt to terminate if it exceeds the deadline.

2 Specify the number of pods to run concurrently.

3 The spec includes a pod template.

4 Specify the name of the service account to associate with the pod. If you do not specify a service account, then the pod will use the default service account in the namespace.

### Scheduling OpenShift Cron Jobs
CronJobs resources create jobs based on a given time schedule. Use CronJobs for running automation, such as backups or reports, which should be executed on a regular interval.

~~~
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cron
  namespace: database
spec:
  schedule: "1 0 * * *" 
  jobTemplate: 
    spec:
      activeDeadlineSeconds: 600
      parallelism: 2
      template:
        metadata:
          name: backup
        spec:
          serviceAccountName: backup_sa
          containers:
          - name: backup
            image: example/backup_maker:v1.2.3
          restartPolicy: OnFailure
~~~

### Navigating the OpenShift REST API

**Authenticating with the REST API**
For interacting with the OpenShift API, two security concepts intervene:

  - Authentication. Handled by the OpenShift OAuth server, its purpose is to validate that users are who they say they are.

  - Authorization. Handled by the OpenShift API server, its purpose is to validate that users have access to the resources they are trying to interact with.

In order to interact with OpenShift resources via the REST API, you must retrieve a bearer token from the OpenShift OAuth server, and then include this token as a header in requests to the API server.

There are several methods for retrieving the token, including:

  1. Request a token from the OAuth server path /oauth/authorize?client_id=openshift-challenging-client&response_type=token. The server responds with a 302 Redirect to a location. Find the access_token parameter in the location query string.

  2. Log in using the oc login command and inspect the kubeconfig YAML file. This is normally located at ~/.kube/config. Find the token listed under your user entry.

  3. Log in using the oc login command, and then run the oc proxy command to expose an API server proxy on your local machine. Any requests to the proxy server will identify as the user logged in with oc.

  4. Log in using the oc login command, and then run the command oc whoami -t.

How to grab the token from the `API` server.

Find the oauth server.
~~~
$ oc get route -n openshift-authentication
NAME             HOST/PORT                              PATH  SERVICES ...
oauth-openshift  oauth-openshift.apps.ocp4.example.com        oauth-openshift ...
~~~
When you have the `HOST` from above, run the command below. Look for the line that starts with `Location`.

~~~
$ curl -u <user> -kv "https://oauth-openshift.apps.ocp4.example.com/oauth/authorize?client_id=openshift-challenging-client&response_type=token"

...output omitted...
< Location: https://oauth-openshift.apps...example.com/oauth/token/implicit#access_token=sha256~xvZ8SsTiA3jRIiEX9QMUOLdaZRUPqubLy2AiQbQGDb0
&expires_in=86400&scope=user%3Afull&token_type=Bearer
...output omitted...
~~~

After that, you must include the bearer token as a header in requests to the API server as follows:

~~~
$ curl -k \
  --header "Authorization: Bearer sha256~Ylfa...8sOY" \
  -X GET https://api.example.com:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.136.182:6443"
    }
  ]
}
~~~

**Finding REST API Paths**

The OpenShift REST API paths can be explored with a HTTP GET request to retrieve a list of possible paths:

~~~
$ curl -k \
  --header "Authorization: Bearer sha256~Ylfa...8sOY" \
  -X GET https://api.example.com:6443/openapi/v2 | jq
{
  "swagger": "2.0",
  "info": {
    "title": "Kubernetes",
    "version": "v1.23.3+e419edf"
  },
  "paths": {
    ...output omitted...
    "/api/v1/pods": {
      ...output omitted...
    }
    ...output omitted...
    "/apis/rbac.authorization.k8s.io/v1/clusterrolebindings": {
      ...output omitted...
    }
    ...output omitted...
    "/apis/apps/v1/namespaces/{namespace}/deployments": {
      ...output omitted...
    }
    ...output omitted...
    "/apis/route.openshift.io/v1/namespaces/{namespace}/routes": {
      ...output omitted...
    }
  }
...output omitted...
}
~~~

### Writing Ansible Playbooks to Manage OpenShift Resources

Ansible Content for Kubernetes and OpenShift clusters: 'kubernetes.core' and 'redhat.openshift'.

Using modules for k8s or OCP requires you authenticate to the API. There several methods to do so. 

K8s and OCP have common parameters.
  - api_key
  - host
  - ca_cert
  - namespace
 Use module defaults.
~~~
---
- name: Configuring the OpenShift cluster
  hosts: localhost
  module_defaults:
    group/kubernetes.core.k8s:
      api_key: "{{ auth_token }}"
      host: https://api.example.com:6443
      ca_cert: /etc/pki/tls/certs/ca-bundle.crt
    group/redhat.openshift.openshift:
      api_key: "{{ auth_token }}"
      host: https://api.example.com:6443
      ca_cert: /etc/pki/tls/certs/ca-bundle.crt
...output omitted...
~~~
Example playbook
~~~
---
- name: Logging in to OpenShift
  hosts: localhost

  tasks:
    - name: Ensure an access token is retrieved for the developer user
      redhat.openshift.openshift_auth:  1
        host: https://api.example.com:6443
        username: developer
        password: developer
      register: auth_results  2

- name: Deploying the intranet front end application
  hosts: localhost

  module_defaults:  3
    group/redhat.openshift.openshift:
      namespace: intranet-front
      api_key: "{{ auth_results['openshift_auth']['api_key'] }}"
      host: https://api.example.com:6443
    group/kubernetes.core.k8s:
      namespace: intranet-front
      api_key: "{{ auth_results['openshift_auth']['api_key'] }}"
      host: https://api.example.com:6443

  tasks:
    - name: Ensure the project exists
      redhat.openshift.k8s:
        state: present
        resource_definition:  4
          apiVersion: project.openshift.io/v1
          kind: Project
          metadata:
            name: intranet-front

    - name: Ensure the intranet front end is deployed
      redhat.openshift.k8s:
        state: present
        src: intranet-front.yml  5

    - name: Ensure the deployments is scaled up
      kubernetes.core.k8s_scale:  6
        kind: Deployment
        name: intranet-front
        replicas: 5

    - name: Ensure a route exists
      redhat.openshift.openshift_route:  7
        service: intranet-front-svc
      register: route

    - name: Ensure the route is displayed
      debug:
        msg: "The Intranet is available at
              http://{{ route['result']['spec']['host'] }}"
~~~
# Managing OpenShift Operators
### Describing Operators

OpenShift operators implement OpenShift features such as self-healing, updates, and other
administrative tasks, either on resources or cluster-wide actions. Operators package, deploy, and
manage an OpenShift application.

OpenShift uses operators to manage the cluster by controlling tasks, such as upgrades and
self-healing, among others. For example, the Cluster Version Operator (CVO) manages cluster
operators and their upgrades.
Additional operators can run applications or add features to OpenShift.
There are several operators in an OpenShift cluster, such as:
  - The OperatorHub, which is a registry for OpenShift operators.
  - The Operator Lifecycle Manager (OLM), which installs, updates, and manages operators.

Use either the CLI or the web console to install operators located in the OperatorHub.

### Installing Operators from the OperatorHub

There are several ways to install operators in an OpenShift cluster, including: OperatorHub, Helm
charts, and custom YAML files, among others. The recommended way to install operators is from
the OperatorHub, using either the Web Console or the CLI. Installing operators from the Web
Console is more straightforward than using the CLI. However, creating resource files and applying
them by using the CLI allows you to automate your installation.

Operators that come with OpenShift (also called cluster operators) are managed by the Cluster
Version Operator, and operators that are installed from the OperatorHub are managed by the
Operator Lifecycle Manager (OLM).

There are three initial settings that must be defined when installing an operator.
Installation mode
  - An operator can be installed on all namespaces or on an individual namespace, if supported.
Update Channel
  - If the operator is available through multiple channels, this option selects the channel to which the operator will subscribe.
Approval strategy
  - This option determines whether the operator is updated, either automatically or manually.

### Installing an Operator from the OLM by Using the CLI

~~~
$ oc get packagemanifests

$ oc describe packagemanifests <operator>

$ oc describe packagemanifests mariadb-operator
~~~

### Install an Operator

Create a namespace for the operator
~~~
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
  name: namespace
~~~

Create an operator group object.
~~~
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operatorgroup-name
  namespace: namespace
spec:
  targetNamespaces:
  - namespace
~~~

Create a subscription.
~~~
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: subscription-name (1)
  namespace: namespace (2)
spec:
  channel: "4.7"
  name: file-integrity-operator (3)
  source: redhat-operators (4)
  sourceNamespace: openshift-marketplace (5)
~~~

1. Name of the subscription
2. Namespace in which the operator will run
3. Name of the operator to subscribe to
4. Catalog source that provides the operator
5. Namespace of the catalog source

Check the logs to verify opertor installation.
~~~
$ oc logs pod/olm-operator-c5599dfd7-nknfx -n openshift-operator-lifecycle-manager
~~~

Inspect the subscription object to verify the operator status.
~~~
$ oc describe sub `subscription-name` -n `namespace`
~~~

List All Operatots
~~~
$ oc get csv -A
~~~

Operators can be subscribed to one namespace or to all namespaces. To list the operators
managed by the OLM, list the active subscriptions.
~~~
$ oc get subs -A
~~~

To view the status and events from custom resources related to a given operator, describe the
operator deployment.
~~~
$ oc describe deployment.apps/file-integrity-operator |grep -i kind
~~~

Check the operator elements by getting all objects in the operator’s namespace. If the operator is installed in all the namespaces, then make the query in the openshift-operators namespace and look for the name of the operator to discover the elements.
~~~
$ oc get all -n openshift-file-intergrity
~~~

Use the oc logs command to view Events and logs of an operator.
~~~
$ oc logs deployment.apps/file-integrity-operator
~~~

Modify an Operator fromthe OLM Usinh the CLI
~~~
$ oc apply -f file-integrity-operator-subscription.yaml
~~~

### Deleting the Subscription and Cluster Service Version objects
Review the current version of the subscribed operator in the currentCSV field.
~~~
$ oc get sub `subscription-name` -o yaml | grep currentCSV
$ oc delete sub `currentCSV`
$ oc delete csv `currentCSV`
~~~

### Describing Cluster Operators
~~~
$ oc get clusteroperator
NAME                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication             4.10.20    True        False         False      54m
baremetal                  4.10.20    True        False         False      98d
cloud-controller-manager   4.10.20    True        False         False      98d
cloud-credential           4.10.20    True        False         False      98d
~~~

### Describing the Cluster Version Operator
Run the `oc get clusterversion version` command to retrieve the release image that the
CVO uses:
~~~
$ oc get clusterversion version -o jsonpath='{.status.desired.image}'
quay.io/openshift-release-dev/ocp-release@sha256:7ffe...cc56
~~~

Extract the contents of the release image to a local directory:
~~~
$ oc adm release extract --to=release-image --from=quay.io/openshift-release-dev/ocp-release@sha256:7ffe...cc56
~~~

The following example displays the details of the Cluster Samples Operator, which manages the image streams and the templates i Authentication and authorizationn the `openshift` namespace
~~~
$ grep -l "kind: ClusterOperator" release-image/*
...output omitted...
release-image/0000_50_cluster-samples-operator_07-clusteroperator.yaml
...output omitted...

$ cat release-image/0000_50_cluster-samples-operator_07*.yaml
apiVersion: config.openshift.io/v1
kind: ClusterOperator
metadata:
name: openshift-samples
...output omitted...
~~~

# GitOps

**Deploying Jenkins**
Create project
deploy jenkins
add self-provisioner to the sa
check the logs of the running pods to make sure its up and running
Get the address
~~~
$ oc new-project jenkins
$ oc new-app --template jenkins-persistent
$ oc adm policy add-cluster-role-to-user self-provisioner -z jenkins -n jenkins
$ oc get pods 
$ oc logs jenkins-x-xxxx |grep 'up and running'
$ oc get routes
~~~

# Configuring Enterprise Authentication
### Configuring the LDAP Identity Provider

1. Create secret for the bind password
~~~
$ oc create secret generic ldap-secret -n openshift-config --from-literal=bindPassword=$(LDAP_ADMIN_PASSWORD)
~~~

2. Configure cert if using TLS.
~~~
$ oc create secret configmap ca-config-map -n openshift-config --from-file=ca.crt
~~~

3. Update OpenShift oauth configuration for LDAP.
~~~
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ldapidp
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: "uid=admin,cn=users,cn=accounts,dc=ocp4,dc=example,dc=com"
      bindPassword: 
        name: ldap-secret
      ca:
        name: ca-config-map
      insecure: false
      url: "ldaps://idm.ocp4.example.com/cn=users,cn=accounts,dc=ocp4,dc=example,dc=com?uid" 
~~~

4. Wait for OpenShift authentications to recreate then test.

5. Add permissions to users.
~~~
$ oc adm policy add-cluster-role-to-user cluster-admin admin
~~~

### Syncing LDAP Groups

1. Create the LDAP sync config.
~~~
kind: LDAPSyncConfig
apiVersion: v1
url: ldaps://idm.ocp4.example.com
bindDN: uid=ldap_user_for_sync,cn=users,cn=accounts,dc=example,dc=com
bindPassword: ldap_user_for_sync_password
insecure: false
ca: /path/to/ca.crt
rfc2307: 5
    groupsQuery:
        baseDN: "cn=groups,cn=accounts,dc=example,dc=com"
        scope: sub
        derefAliases: never
        pageSize: 0
        filter: (objectClass=posixgroup)
    groupUIDAttribute: dn
    groupNameAttributes: [ cn ]
    groupMembershipAttributes: [ member ]
    usersQuery:
        baseDN: "cn=accounts,dc=example,dc=com"
        scope: sub
        derefAliases: never
        pageSize: 0
    userUIDAttribute: dn
    userNameAttributes: [ cn ]
    tolerateMemberNotFoundErrors: false
    tolerateMemberOutOfScopeErrors: true
~~~

2. Test
~~~
$ oc adm groups sync --sync-config ldap-sync.yml
~~~

3. Create a project
~~~
$ oc new-project syncldap
~~~

4. Create serviceaccount, ClusterRole, and clusterRoleBinding
5. Create configMap for cert and secret for bindPassword
6. Create configMap for the ldap-sync
7. Create cron job for to sync. 
8. Check the logs

# Configuring Trusted Certificates
### Changing the Ingress Controller Operator Certificate

1. Create a new configMap in the `openshift-config` project
~~~
$ oc create configmap <CONFIGMAP-NAME> --from-file ca-bundle.crt=<PATH-TO-CERTIFICATE> -n openshift-config
~~~

2. Configure the proxy to use the new configmap
~~~
$ oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"<CONFIGMAP-NAME>"}}}'
~~~

3. Create a new TLS secret in the openshift-ingress namespace using the new certificate and its corresponding key. 
~~~
$ oc create secret tls <SECRET-NAME> --cert <PATH-TO-CERTIFICATE> --key <PATH-TO-KEY> -n openshift-ingress
~~~

4. 


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


# Monitoring and Metrics

Assigning Cluster Monitoring Roles

~~~
$ oc adm policy add-cluster-role-to-user cluster-monitoring-view USER
~~~

### Describing Alertmanager Features

|State|Description|
|---|---|
|Firing|  The alert rule evaluates to true, and has evaluated to true for longer than the defined alert duration.|
|Pending | The alert rule evaluates to true, but has not evaluated to true for longer than the defined alert duration.|
|Silenced | The alert is Firing, but is actively being silenced. Administrators can silence an alert to temporarily deactivate it.
|Not Firing |  Any alert that is not Firing, Pending, or Silenced is labeled as Not Firing.

### Describing Alertmanager Default Receivers

By default, alerts are not sent to external locations. The OpenShift web console displays alerts at
Observe > Alerting. Also, alerts are accessible from the Alertmanager API.

~~~
[user@host ~]$ ALERTMANAGER="$(oc get route/alertmanager-main -n openshift-monitoring -o jsonpath='{.spec.host}')"
[user@host ~]$ curl -s -k -H "Authorization: Bearer $(oc sa get-token prometheus-k8s -n openshift-monitoring)" https://${ALERTMANAGER}/api/v1/ alerts | jq .
~~~
Configure Alertmanager to send alerts to external locations, such as email, PagerDuty, and HipChat, to promptly notify you about cluster problems. Alertmanager sends alerts to the locations configured in the alertmanager-main secret in the openshift-monitoring namespace.
This is the default configuration of the alertmanager-main secret. The double quotes can be removed to improve readability.

~~~
"global":
"resolve_timeout": "5m"
"receivers":
- "name": "null"
"route":
"group_by":
- "namespace"
"group_interval": "5m"
"group_wait": "30s"
"receiver": "null"
"repeat_interval": "12h"
"routes":
- "match":
"alertname": "Watchdog"
"receiver": "null"
~~~
This will have `"` in it and they must be removed before applying to the cluster. 
~~~
$ sed -i 's/"//g' /tmp/alertmanager.yaml
~~~

Email Example:
~~~
"global":
  "resolve_timeout": "5m"
  "smtp_smarthost": "utility.lab.example.com:25"
  "smtp_from": "alerts@ocp4.example.com"
  "smtp_auth_username": "smtp_training"
  "smtp_auth_password": "Red_H4T@!"
"  smtp_require_tls": false
"receivers":
- "name": "email-notification"
  "email_configs":
    - "to": "ocp-admins@example.com"
- "name": "default"
"route":
  "group_by":
  - "job"
  "group_interval": "5m"
  "group_wait": "30s"
  "receiver": "default"
  "repeat_interval": "12h"
  "routes":
  - "match":
      "alertname": "Watchdog"
    "receiver": "default"
  - "match":
      "severity": "critical"
    "receiver": "email-notification"
~~~

###  Applying a New Alertmanager Configuration
  - Apply a new Alertmanager configuration by updating the alertmanager-main secret in the `openshift-monitoring` namespace.
~~~
$ oc set data secret/alertmanager-main -n openshift-monitoring --from-file=/tmp/alertmanager.yaml
secret/alertmanager-main data updated

$ oc logs -f -c alertmanager alertmanager-main-0 -n openshift-monitoring
...output omitted...
level=info ts=2021-10-15T18:53:09.052Z caller=coordinator.go:119
component=configuration msg="Loading configuration file" file=/etc/alertmanager/
config/alertmanager.yaml
level=info ts=2021-10-15T18:53:09.052Z caller=coordinator.go:131
component=configuration msg="Completed loading of configuration file" file=/etc/
alertmanager/config/alertmanager.yaml
~~~

### Troubleshooting Using the Cluster Monitoring Stack

  - Prometheus is an open source project for system monitoring and alerting.
  - Both Red Hat OpenShift Container Platform and Kubernetes integrate Prometheus to enable cluster metrics, monitoring, and alerting capabilities.

  - Prometheus gathers and stores streams of data from the cluster as time-series data. Time-series data consists of a sequence of samples, with each sample containing:
    - A timestamp.
    - A numeric value (such as an integer, float, or Boolean).
    - A set of labels in the form of key/value pairs. The key/value pairs are used to isolate groups of related values for filtering.

For example, the machine_cpu_cores metric in Prometheus contains a sequence of measurement samples of the number of CPU cores for each machine.

**OpenShift integrates Prometheus metrics at `Observe > Metrics`.**

### Reviewing OpenShift Metrics
As mentioned elsewhere in this chapter, OpenShift has three monitoring stack components to gather the metrics from the Kubernetes API: the `kube-state-metrics`, `openshift-state- metrics`, and `node-exporter` agents.

### Describing Prometheus Query Language
Prometheus provides a query language, PromQL, that allows you to select and aggregate time-series data.
You can filter a metric to include only certain key/value pairs. For example, modify the previous query to show only metrics for the worker02 node using the following expression:

~~~
instance:node_cpu_utilisation:rate1m{instance="worker02"}
~~~
`sum()`
  - Totals the value of all sample entries at a given time
`rate()`
  - Computes the per-second average of a time series for a given time range
`count()`
  - Counts the number of sample entries at a given time
`max()` 
  - Select the maximum over the sample entries.

The following are two examples of Prometheus Query Language expressions that use one metric gathered by the node-exporter agent, and another metric gathered by the `kube-state-metrics` agent .

~~~
node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes*100<50
  Shows nodes with less than 50% of memory available.
kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
  Shows persistent volume claims in pending state.
~~~

OpenShift cluster monitoring includes the following default dashboards:

etcd
  - This dashboard provides information on etcd instances running in the cluster.

Kubernetes/Compute Resources/Cluster
  - This dashboard provides a high-level view of cluster resources.

Kubernetes/Compute Resources/Namespace (Pods)
  - This dashboard displays resource usage for pods within a namespace.

Kubernetes/Compute Resources/Namespace (Workloads)
  - This dashboard filters resource usage first by namespace and then by workload type, such as deployment, daemon set, and stateful set. This dashboard displays all workloads of the specified type within the namespace.

Kubernetes/Compute Resources/Node (Pods)
  - This dashboard shows pod resource usage filtered by node.

Kubernetes/Compute Resources/Pod
  - This dashboard displays the resource usage for individual pods. Select a namespace and a pod within the namespace.

Kubernetes/Compute Resources/Workload
  - This dashboard provides resources usage filtered by namespace, workload, and workload type.

Kubernetes/Networking/Cluster
  - This dashboard displays network usage for the cluster, and sorts many items to show namespaces with the highest usage.

Kubernetes/Networking/Namespace (Pods)
  - This dashboard displays network usage for pods within a namespace.

Kubernetes/Networking/Pod
  - This dashboard displays the network usage for individual pods. Select a namespace and a pod within the namespace.

Prometheus
  - This dashboard provides detailed information about the prometheus-k8s pods running in the openshift-monitoring namespace.

USE Method/Cluster
  - USE is an acronym for Utilization Saturation and Errors. This dashboard displays several graphics that can identify if the cluster is over-utilized, over-saturated, or experiencing a large number of errors. Because the dashboard displays all nodes in the cluster, you might be able to identity a node that is not behaving the same as the other nodes in the cluster.

### Configuring Storage for the Cluster Monitoring Stack

After installing the cluster monitoring stack, you can configure persistent storage to prevent monitoring data loss. This configuration enables you to keep a record of the past cluster status that you can use to investigate and correlate current and past issues within the cluster.

The following cluster monitoring operator components are configurable
  - alertmanagerMain
  - k8sPrometheusAdapter
  - kubeStateMetrics
  - nodeExporter
  - openshiftStateMetrics
  - prometheusK8s
  - prometheusOperator
  - thanosQuerier
  - telemeterClient

The basic skeleton for the cluster-monitoring-config configuration map follows
~~~
apiVersion: 1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
   <component>:
     <component-configuration-options>
~~~

~~~
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    prometheusK8s:
      retention: 15d
      volumeClaimTemplate:
        metadata:
          name: prometheus-local-pv
        spec:
          storageClassName: gp2
          volumeMode: Filesystem
          resources:
            requests:
            storage: 40Gi
~~~

---
# Provisioning and Inspecting Cluster Logging

### Red Hat OpenShift Logging Components
logStore
  - The logStore is the Elasticsearch cluster that:
  - Stores the logs into indexes.
  - Provides RBAC access to the logs.
  - Provides data redundancy.

collection
  - Implemented with Fluentd, the collector collects node and application logs, adds pod and namespace metadata, and stores them in the logStore. The collector is a DaemonSet, so there is a Fluentd pod on each node.

visualization
  - The centralized web UI from Kibana displays the logs and provides a way to query and chart the aggregated data. 

event routing
  - The Event Router monitors the OpenShift events API and sends the events to STDOUT so the collector can forward them to the logStore. The events from OpenShift are stored in the infra index in Elasticsearch.

### Installing the Elasticsearch Operator

The Elasticsearch Operator handles the creation of and updates to the Elasticsearch cluster defined in the Cluster Logging Custom Resource. Each node in the Elasticsearch cluster is deployed with a PVC named and managed by the Elasticsearch Operator. A unique Deployment object is created for each Elasticsearch node to ensure that each Elasticsearch node has a storage volume of its own.

The Elasticsearch Operator must be installed in a namespace **other than** `openshift-operators` to avoid possible conflicts with metrics from other community operators. Create and use the `openshift-operators-redhat` namespace for the Elasticsearch Operator.

~~~
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-operators-redhat
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-monitoring:"true"
~~~

Create an `OperatorGroup` object to install the operator in all namespaces and a `Subscription` object to subscribe the `openshift-operators-redhat` namespace to the operator.

~~~
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-operators-redhat
  namespace: openshift-operators-redhat
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: elasticsearch-operator
  namespace: openshift-operators-redhat
spec:
  channel: stable
  installPlanApproval: Automatic
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: elasticsearch-operator
~~~

Create the RBAC objects to grant Prometheus permission to access the `openshift-operators-redhat` namespace.
~~~
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: openshift-operators-redhat
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: openshift-operators-redhat
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: openshift-operators-redhat
~~~

---
[Objectives:](https://www.redhat.com/en/services/training/ex380-certified-specialist-openshift-automation-exam?section=objectives) 
Study points for the exam
To help you prepare, the exam objectives highlight the task areas you can expect to see covered in the exam. Red Hat reserves the right to add, modify, and remove exam objectives. Such changes will be made public in advance. 

As part of this exam, you should be able to perform these tasks:

Deploy Kubernetes applications on OpenShift
  - Assemble an application from Kubernetes components
  - Understand and use Kustomize
  - Use an image stream with a Kubernetes deployment
Configure and automate OpenShift tasks
  - Create a simple script to automate a task
  - Deploy an existing script to automate a task
  - Troubleshoot and correct a script
  - Understand and query the REST API using CLI tools
  - Create a custom role
  - Create a cron job
  - Create a simple Ansible playbook
Work with and manage OpenShift Operators
  - Install an operator
  - Update an operator
  - Delete an operator
  - Subscribe an operator
  - Troubleshoot an operator
Work with registries
  - Pull/push content from remote registries
  - Tag images in remote registries
Implement GitOps with Jenkins
  - Deploy a Jenkins master
  - Create a Jenkins pipeline to remediate configuration drift
Configure Enterprise Authentication
  - Configure an LDAP identity provider
  - Configure RBAC for an LDAP provided user account
  - Synchronize OpenShift groups with LDAP
Understand and manage ingress
  - Use the oc route command to expose services
  - Understand how ingress components relate to OpenShift deployments and projects
  - Configure trusted TLS Certificates
  - Work with certificates using the web and CLI interfaces
  - Renew and apply a certificate
Work with machine configurations
  - Understand MachineConfig object structure
  - Create custom machine configurations
Configure Dedicated Node Pools
  - Add a worker node
  - Create custom machine config pools
Configure Persistent Storage
  - Provision shared storage for applications
  - Provision block storage
  - Configure and use storage quotas, classes, and policies
  - Troubleshoot storage issues
Manage Cluster Monitoring and Metrics
  - Manage OpenShift alerts
  - Use monitoring to troubleshoot cluster issues
Provision and Inspect Cluster Logging
  - Deploy cluster logging
  - Query cluster logs
  - Diagnose cluster logging problems
Recover Failed Worker Nodes
  - Diagnose worker node failures
  - Recover a node that has failed