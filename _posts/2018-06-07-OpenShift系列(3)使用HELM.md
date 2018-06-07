---
tags:
- OpenShift
- Kubernetes
- Paas
- DevOp
- Docker
- helm
key: 20180607
---
# OpenShift系列(3)使用HELM
先写了英文文档，实在没有时间再写中文的了，先贴出来。有人需要用自取，等有时间再写中文的
<!--more-->

# Helm on OpenShift

## Overview
Helm is the package manager for Kubernetes. As a wrapper of Kubernetes the OpenShift Origin can use Helm as well. Because there are more access controls on OpenShift origin cluster. We may need some manual operations to make Helm working on OpenShift Origin cluster.

**Make sure you have basic knowledge on Helm and OpenShift**

## Install

Helm cli use tiller installed on Kube cluster to operate objects. For OpenShift the tiller should be installed under certain project(namespace)
```
            |  OpenShift Cluster
            |----------------------
            | Project 1
Helm CLI ---|---> Tiller
            |-----------------------
            | Project 2
```
So install Helm for OpenShift including
* Setup binary in local
* Create `project`, `service account`, `role` for tiller and binding access role
* Install tiller on Cluster and binding with service account created

### Setup binary in local
For Linux server you can download and setup with command below:
```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.9.0-linux-amd64.tar.gz | tar xz
$ cd linux-amd64
$ ./helm init --client-only
```
Now you have the Helm CLI ready in local.

### Create objects needed on cluster for Tiller
Plan the project(namespace) name you want to install the tiller. You can create that by certain user (better be system:admin) and hide to others to make sure no exceptional operations made to the tiller. Please pay attention that the helm chart history will be saved as configmaps in that project.

 In this example I will use `tiller` as the project name and the same with `role` and `service account` name.

Some of the actions need cluster admin role. So you'd better login with system:admin `oc login -u system:admin`

***Project & Service account*** -- creating
```
#Create project tiller
oc new-project tiller
#Create service account
oc create serviceaccount tiller
```
***Role*** -- Save the content below to a file like `roleTiller.yaml`. This is an example role to make the tiller role can access configmaps and list namespaces. That's the basic access requirement for normal tiller operations.
```yaml
apiVersion: v1
items:
- apiVersion: authorization.openshift.io/v1
  kind: Role
  metadata:
    name: tiller
  rules:
  - apiGroups:
    - ""
    resources:
    - configmaps
    verbs:
    - create
    - delete
    - get
    - list
    - update
  - apiGroups:
    - ""
    resources:
    - namespaces
    verbs:
    - get
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
Then run command below to create role
```
#Create role tiller
oc create -f roleTiller.yaml
#Verify
oc get role
```
***RoleBinding*** -- run command below to bind the service account tiller with role tiller
```
#Binding service account to role
oc adm policy add-role-to-user tiller -z tiller
#verify
oc get rolebinding tiller -o yaml
```
Now you have a service account ready for tiller use. You can install tiller now.

### Install tiller

We will install the tiller as a deployment which with 1 pod running tiller docker image. Because for the time I write this artical the helm in in V2.9.0. So using 2.9.0 as the image version.

Save the yaml content below in file `deploymentTiller.yaml`
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: tiller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helm
      name: tiller
  template:
    metadata:
      labels:
        app: helm
        name: tiller
    spec:
      containers:
        image: gcr.io/kubernetes-helm/tiller:v2.9.0
        name: tiller
        livenessProbe:
          httpGet:
            path: /liveness
            port: 44135
        ports:
        - containerPort: 44134
          name: tiller
        readinessProbe:
          httpGet:
            path: /readiness
            port: 44135
      serviceAccountName: tiller
```
Install the deployment in server with command
```
oc create -f deploymentTiller.yaml
```

### Set helm cli namespace and verify
The OpenShift project is equals to the Kubenetes namespace. So that when you install the tiller to the project `tiller`. It means the tiller is installed in namespace `tiller`. You can specify the namespace to helm with parameter ` --tiller-namespace string` or set that for default context with export envirnment varilable `TILLER_NAMESPACE`. Now I will set that with environment variable and try to verify the cli and server connection.
```
#Set namespace
export TILLER_NAMESPACE=tiller
#Run version to verify
helm version
```
If the install successful. You may receive the resposne as below
```
Client: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
```

## Day two managements for Helm on OpenShift

After install the helm. We can start deploy things with helm chart. In this section I will proivde some sample for deployment chart and how to add extra access for Helm tiller service account to finish some special operations needed by chart.

### Sample for create new project and assign edit access to helm tiller for thar project

To create new project (namespace) and deploy helm chart. You should do steps below
1. Make sure the tiller installed correctly
2. Create new Project
3. Adding edit role for tiller service account for this Project
4. Deploy helm chart to the project (If special access needed assign that before)

```
#Check tiller install
$helm Version
Client: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
#Create new Project
$oc new-project testproject
Now using project "testproject" on server "https://9.30.192.245:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
#Add edit role for tiller service account for this project (You set the tiller namespace by environment varialble TILLER_NAMESPACE already)
$oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"
role "edit" added: "system:serviceaccount:tiller:tiller"
```

For now you are ready to deploy any chart on this project (namespace).
```
$helm install stable/hackmd
NAME:   oldfashioned-ladybird
LAST DEPLOYED: Sun May 20 22:44:07 2018
NAMESPACE: testproject
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                              TYPE    DATA  AGE
oldfashioned-ladybird-postgresql  Opaque  1     0s

==> v1/PersistentVolumeClaim
NAME                              STATUS   VOLUME  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
oldfashioned-ladybird-postgresql  Bound    pv0046  100Gi     RWO,ROX,RWX   0s
oldfashioned-ladybird-hackmd      Pending  0s

==> v1/Service
NAME                              TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
oldfashioned-ladybird-postgresql  ClusterIP  172.30.58.53   <none>       5432/TCP  0s
oldfashioned-ladybird-hackmd      ClusterIP  172.30.97.122  <none>       3000/TCP  0s

==> v1beta1/Deployment
NAME                              DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
oldfashioned-ladybird-postgresql  1        0        0           0          0s

==> v1beta2/Deployment
oldfashioned-ladybird-hackmd  1  0  0  0  0s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace testproject -l "app=hackmd,release=oldfashioned-ladybird" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80

```

Now you received a hackmd deployment. You may receive error for postgresl pod for
`initdb: could not look up effective user ID 1000100000: user does not exist`.
That's caused by Openshift doesn't allow container to be ran with root user. You can update the policy as
```
$oc adm policy add-scc-to-user anyuid -z default
scc "anyuid" added to: ["system:serviceaccount:testproject:default"]
```
Then scale down/up the deployment. You will made the whole enviornment working.
```
$oc get all
NAME                                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/oldfashioned-ladybird-hackmd       1         1         1            1           8m
deploy/oldfashioned-ladybird-postgresql   1         1         1            1           8m

NAME                                             DESIRED   CURRENT   READY     AGE
rs/oldfashioned-ladybird-hackmd-7cd7b9788d       1         1         1         8m
rs/oldfashioned-ladybird-postgresql-6995bb54b9   1         1         1         8m

NAME                                                   READY     STATUS    RESTARTS   AGE
po/oldfashioned-ladybird-hackmd-7cd7b9788d-2p8bk       1/1       Running   0          49s
po/oldfashioned-ladybird-postgresql-6995bb54b9-lpxw8   1/1       Running   0          1m

NAME                                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
svc/oldfashioned-ladybird-hackmd       ClusterIP   172.30.97.122   <none>        3000/TCP   8m
svc/oldfashioned-ladybird-postgresql   ClusterIP   172.30.58.53    <none>        5432/TCP   8m
```
You can list the chart installed by Using
```
$helm list
NAME                    REVISION        UPDATED                         STATUS          CHART           NAMESPACE
oldfashioned-ladybird   1               Sun May 20 22:44:07 2018        DEPLOYED        hackmd-0.1.0    testproject
```

You can uninstall the chart installed by Using
```
$helm delete oldfashioned-ladybird
release "oldfashioned-ladybird" deleted
```

### Adding extra access for Tiller

For most case the created `tiller` role combine with `edit` role is enough for tiller to operate chart in project. But in openshift some of the resource is considered as cluster resource. So that you cannot operate that with project role with edit access. The most important one is `PersistentVolumes`. If the chart contains `PersistentVolumes`. Then with the default roles the install will failed when create PersistentVolumes.

For OpenShift. The persistentVolumes are considered as cluster resource. should have the cluster role of `cluster-admin` or `storage-admin`. For `cluster-admin` it's too large who can control all in cluster. So `storage-admin` is OK. You can assign the role to user with command
```
$oc adm policy add-cluster-role-to-user storage-admin "system:serviceaccount:${TILLER_NAMESPACE}:tiller"
cluster role "storage-admin" added: "system:serviceaccount:tiller:tiller"
```
If you meet any access issue for tiller operations. You can check the role needed and assign to tiller service account followed this way.
