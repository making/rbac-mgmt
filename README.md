# RBAC Management for K8s

## How to install RBAC Management for K8s


### Manage RBAC using Tanzu CLI

Prerequisite:
* [kapp-controller](https://carvel.dev/kapp-controller/)

Add the Package Repository:

```
NAMESPACE=<namespace to install the package>

tanzu package repository add rbac-mgmt-repository -n ${NAMESPACE} --url ghcr.io/making/rbac-mgmt-repo:0.0.2
```

List the available packages to confirm the addition:

```
$ tanzu package available list -n ${NAMESPACE} 

  NAME                    DISPLAY-NAME  SHORT-DESCRIPTION  LATEST-VERSION         
  rbac-mgmt.pkg.maki.lol  rbac-mgmt     RBAC Management    0.0.1 
  rbac-mgmt.pkg.maki.lol  rbac-mgmt     RBAC Management    0.0.2 
```

Check the values for the RBAC Management package:

```
$ tanzu package available get -n ${NAMESPACE} rbac-mgmt.pkg.maki.lol/0.0.2 --values-schema

  KEY                                 DEFAULT  TYPE   DESCRIPTION                                            
  clusterrolebinding.groups           []       array  Definition of ClusterRoleBinding for groups            
  clusterrolebinding.serviceaccounts  []       array  Definition of ClusterRoleBinding for service accounts  
  clusterrolebinding.users            []       array  Definition of ClusterRoleBinding for users             
  rolebindings                        []       array  Definition of RoleBindings per namespace  
```

For the more detailed schema, see [the openapi spec](./repo/packages/rbac-mgmt.pkg.maki.lol/0.0.2.yaml)

Install the RBAC Management package, using the values file:

```yaml
cat <<EOF > rbac-mgmt-values.yaml
---
rolebindings:
- namespace: default
  users:
  - name: peter@example.com
    clusterroles: [ admin ]
- namespace: project-x
  create_namespace: true
  users:
  - name: alex@example.com
    clusterroles: [ edit ]
  - name: oliver@example.com
    clusterroles: [ view ]
  - name: peter@example.com
    clusterroles: [ admin ]
  groups:
  - name: administrators
    clusterroles: [ cluster-admin ]
  serviceaccounts:
  - name: ci-bot
    clusterroles: [ edit ]
EOF

tanzu package install rbac-mgmt -n ${NAMESPACE} -p rbac-mgmt.pkg.maki.lol -v 0.0.2 -f rbac-mgmt-values.yaml
```

Verify PackageInstall has been created:

```
$ tanzu package installed get -n ${NAMESPACE} rbac-mgmt        

NAME:                    rbac-mgmt
PACKAGE-NAME:            rbac-mgmt.pkg.maki.lol
PACKAGE-VERSION:         0.0.2
STATUS:                  Reconcile succeeded
CONDITIONS:              [{ReconcileSucceeded True  }]
```

Verify the created resources:

```yaml
$ kubectl get app -n ${NAMESPACE} rbac-mgmt -oyaml
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  creationTimestamp: "2022-06-09T10:21:15Z"
  finalizers:
  - finalizers.kapp-ctrl.k14s.io/delete
  generation: 1
  name: rbac-mgmt
  namespace: kapp
  ownerReferences:
  - apiVersion: packaging.carvel.dev/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: PackageInstall
    name: rbac-mgmt
    uid: a559ddad-18d2-4add-865e-dfb059cd07ae
  resourceVersion: "145832949"
  uid: eb34cf5c-3c6c-4e9b-851a-e1f308e38534
spec:
  deploy:
  - kapp:
      inspect:
        rawOptions:
        - --tree=true
      rawOptions:
      - --wait-timeout=5m
      - --diff-changes=true
  fetch:
  - imgpkgBundle:
      image: ghcr.io/making/rbac-mgmt-bundle@sha256:9be53eeaff24345d292f38fc4c5037d8a08496183369b8ac41b36f0569dec8cb
      secretRef:
        name: rbac-mgmt-fetch-0
  serviceAccountName: rbac-mgmt-kapp-sa
  syncPeriod: 10m0s
  template:
  - ytt:
      valuesFrom:
      - secretRef:
          name: rbac-mgmt-kapp-values
  - kbld:
      paths:
      - '-'
      - .imgpkg/images.yml
status:
  conditions:
  - status: "True"
    type: ReconcileSucceeded
  consecutiveReconcileSuccesses: 1
  deploy:
    exitCode: 0
    finished: true
    startedAt: "2022-06-09T10:21:19Z"
    stdout: |-
      Target cluster 'https://100.64.0.1:443' (nodes: jaguchi-control-plane-hctz6, 8+)
      @@ create namespace/project-x (v1) cluster @@
            0 + apiVersion: v1
            1 + kind: Namespace
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654770079912481700"
            5 +     kapp.k14s.io/association: v1.f70f8b6fb2967b86812536b39d80020b
            6 +   name: project-x
            7 +
      @@ create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: default @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654770079912481700"
            5 +     kapp.k14s.io/association: v1.c13237b7c67cde32294192f2c1a9d551
            6 +   name: clusterrole-admin-binding
            7 +   namespace: default
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: admin
           12 + subjects:
           13 + - kind: User
           14 +   name: peter@example.com
           15 +
      @@ create rolebinding/clusterrole-edit-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654770079912481700"
            5 +     kapp.k14s.io/association: v1.34e8a6791f6ff4053c4fc3c9080c430c
            6 +   name: clusterrole-edit-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: edit
           12 + subjects:
           13 + - kind: User
           14 +   name: alex@example.com
           15 + - kind: ServiceAccount
           16 +   name: ci-bot
           17 +   namespace: null
           18 +
      @@ create rolebinding/clusterrole-view-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654770079912481700"
            5 +     kapp.k14s.io/association: v1.e317933df089827627ab50148e62f214
            6 +   name: clusterrole-view-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: view
           12 + subjects:
           13 + - kind: User
           14 +   name: oliver@example.com
           15 +
      @@ create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654770079912481700"
            5 +     kapp.k14s.io/association: v1.19b6bf6062f0c08134ca05b009daacb5
            6 +   name: clusterrole-admin-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: admin
           12 + subjects:
           13 + - kind: User
           14 +   name: peter@example.com
           15 +
      @@ create rolebinding/clusterrole-cluster-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654770079912481700"
            5 +     kapp.k14s.io/association: v1.16d7880c92365169f138c22019960642
            6 +   name: clusterrole-cluster-admin-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: cluster-admin
           12 + subjects:
           13 + - kind: Group
           14 +   name: administrators
           15 +
      Changes
      Namespace  Name                               Kind         Conds.  Age  Op      Op st.  Wait to    Rs  Ri
      (cluster)  project-x                          Namespace    -       -    create  -       reconcile  -   -
      default    clusterrole-admin-binding          RoleBinding  -       -    create  -       reconcile  -   -
      project-x  clusterrole-admin-binding          RoleBinding  -       -    create  -       reconcile  -   -
      ^          clusterrole-cluster-admin-binding  RoleBinding  -       -    create  -       reconcile  -   -
      ^          clusterrole-edit-binding           RoleBinding  -       -    create  -       reconcile  -   -
      ^          clusterrole-view-binding           RoleBinding  -       -    create  -       reconcile  -   -
      Op:      6 create, 0 delete, 0 update, 0 noop, 0 exists
      Wait to: 6 reconcile, 0 delete, 0 noop
      10:21:20AM: ---- applying 2 changes [0/6 done] ----
      10:21:20AM: create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: default
      10:21:20AM: create namespace/project-x (v1) cluster
      10:21:20AM: ---- waiting on 2 changes [0/6 done] ----
      10:21:20AM: ok: reconcile rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: default
      10:21:20AM: ok: reconcile namespace/project-x (v1) cluster
      10:21:20AM: ---- applying 4 changes [2/6 done] ----
      10:21:20AM: create rolebinding/clusterrole-cluster-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: create rolebinding/clusterrole-edit-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: create rolebinding/clusterrole-view-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: ---- waiting on 4 changes [2/6 done] ----
      10:21:20AM: ok: reconcile rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: ok: reconcile rolebinding/clusterrole-view-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: ok: reconcile rolebinding/clusterrole-cluster-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: ok: reconcile rolebinding/clusterrole-edit-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      10:21:20AM: ---- applying complete [6/6 done] ----
      10:21:20AM: ---- waiting complete [6/6 done] ----
      Succeeded
    updatedAt: "2022-06-09T10:21:20Z"
  fetch:
    exitCode: 0
    startedAt: "2022-06-09T10:21:15Z"
    stdout: |
      apiVersion: vendir.k14s.io/v1alpha1
      directories:
      - contents:
        - imgpkgBundle:
            image: ghcr.io/making/rbac-mgmt-bundle@sha256:9be53eeaff24345d292f38fc4c5037d8a08496183369b8ac41b36f0569dec8cb
          path: .
        path: "0"
      kind: LockConfig
    updatedAt: "2022-06-09T10:21:19Z"
  friendlyDescription: Reconcile succeeded
  inspect:
    exitCode: 0
    stdout: |-
      Target cluster 'https://100.64.0.1:443' (nodes: jaguchi-control-plane-hctz6, 8+)
      10:21:20AM: info: Resources: Ignoring group version: schema.GroupVersionResource{Group:"stats.antrea.tanzu.vmware.com", Version:"v1alpha1", Resource:"networkpolicystats"}: feature NetworkPolicyStats disabled
      10:21:20AM: info: Resources: Ignoring group version: schema.GroupVersionResource{Group:"stats.antrea.tanzu.vmware.com", Version:"v1alpha1", Resource:"antreanetworkpolicystats"}: feature NetworkPolicyStats disabled
      10:21:20AM: info: Resources: Ignoring group version: schema.GroupVersionResource{Group:"stats.antrea.tanzu.vmware.com", Version:"v1alpha1", Resource:"antreaclusternetworkpolicystats"}: feature NetworkPolicyStats disabled
      Resources in app 'rbac-mgmt-ctrl'
      Namespace  Name                               Kind         Owner  Conds.  Rs  Ri  Age
      project-x  clusterrole-cluster-admin-binding  RoleBinding  kapp   -       ok  -   3s
      project-x  clusterrole-admin-binding          RoleBinding  kapp   -       ok  -   3s
      project-x  clusterrole-edit-binding           RoleBinding  kapp   -       ok  -   3s
      default    clusterrole-admin-binding          RoleBinding  kapp   -       ok  -   3s
      project-x  clusterrole-view-binding           RoleBinding  kapp   -       ok  -   3s
      (cluster)  project-x                          Namespace    kapp   -       ok  -   3s
      Rs: Reconcile state
      Ri: Reconcile information
      6 resources
      Succeeded
    updatedAt: "2022-06-09T10:21:23Z"
  observedGeneration: 1
  template:
    exitCode: 0
    updatedAt: "2022-06-09T10:21:19Z"
```

```
$ kubectl get rolebinding -n project-x -owide 
NAME                                ROLE                        AGE   USERS                GROUPS           SERVICEACCOUNTS
clusterrole-admin-binding           ClusterRole/admin           85s   peter@example.com                     
clusterrole-cluster-admin-binding   ClusterRole/cluster-admin   85s                        administrators   
clusterrole-edit-binding            ClusterRole/edit            85s   alex@example.com                      /ci-bot
clusterrole-view-binding            ClusterRole/view            85s   oliver@example.com 
```

### Manage RBAC using kubectl + ytt
Prerequisite:
* [ytt](https://carvel.dev/ytt/)

```yaml
cat <<EOF > rbac-mgmt-values.yaml
---
rolebindings:
- namespace: default
  users:
  - name: peter@example.com
    clusterroles: [ admin ]
- namespace: project-x
  create_namespace: true
  users:
  - name: alex@example.com
    clusterroles: [ edit ]
  - name: oliver@example.com
    clusterroles: [ view ]
  - name: peter@example.com
    clusterroles: [ admin ]
  groups:
  - name: administrators
    clusterroles: [ cluster-admin ]
  serviceaccounts:
  - name: ci-bot
    clusterroles: [ edit ]
EOF

ytt \
  -f https://github.com/making/rbac-mgmt/raw/main/bundle/config/namespace.yaml \
  -f https://github.com/making/rbac-mgmt/raw/main/bundle/config/rolebinding.yaml \
  -f https://github.com/making/rbac-mgmt/raw/main/bundle/config/clusterrolebinding.yaml \
  -f https://github.com/making/rbac-mgmt/raw/main/bundle/schema.yaml \
  --data-values-file rbac-mgmt-values.yaml \
  | kubectl apply -f-
```

### Manage RBAC using GitOps via App CR
Prerequisite:
* [kapp-controller](https://carvel.dev/kapp-controller/)
* service account (`rbac-mgmt-sa` in the bellow example)
* git repository that manges `rbac-mgmt-values.yaml` (e.g. https://github.com/making/rbac-mgmt-config-example.git)

```yaml
cat <<EOF > app.yaml
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  name: rbac-mgmt
spec:
  serviceAccountName: rbac-mgmt-sa # <--- Needed
  fetch:
  - imgpkgBundle:
      image: ghcr.io/making/rbac-mgmt-bundle:0.0.2
  - git:
      url: https://github.com/making/rbac-mgmt-config-example.git # <--- Change to your repository 
      ref: origin/main
  syncPeriod: 1m
  template:
  - ytt:
      paths:
      - 0/config
      - 0/kapp-config.yaml
      - 0/schema.yaml
      valuesFrom:
      - path: 1/config/rbac-mgmt-values.yaml
  - kbld:
      paths:
      - '-'
      - 0/.imgpkg/images.yml
  deploy:
  - kapp:
      inspect:
        rawOptions:
        - --tree=true
      rawOptions:
      - --wait-timeout=5m
      - --diff-changes=true      
EOF

kubectl apply -f app.yaml -n ${NAMESPACE}
```

Verify the created resources:

```yaml
$ kubectl get app -n ${NAMESPACE} rbac-mgmt -oyaml
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"kappctrl.k14s.io/v1alpha1","kind":"App","metadata":{"annotations":{},"name":"rbac-mgmt","namespace":"kapp"},"spec":{"deploy":[{"kapp":{"inspect":{"rawOptions":["--tree=true"]},"rawOptions":["--wait-timeout=5m","--diff-changes=true"]}}],"fetch":[{"imgpkgBundle":{"image":"ghcr.io/making/rbac-mgmt-bundle:0.0.1"}},{"git":{"ref":"origin/main","url":"https://github.com/making/rbac-mgmt-config-example.git"}}],"serviceAccountName":"kapp","syncPeriod":"1m","template":[{"ytt":{"paths":["0/config","0/schema.yaml"],"valuesFrom":[{"path":"1/config/rbac-mgmt-values.yaml"}]}},{"kbld":{"paths":["-","0/.imgpkg/images.yml"]}}]}}
  creationTimestamp: "2022-06-09T12:37:33Z"
  finalizers:
  - finalizers.kapp-ctrl.k14s.io/delete
  generation: 2
  name: rbac-mgmt
  namespace: kapp
  resourceVersion: "146300783"
  uid: 9db81eda-ff05-469d-a15f-cc766f9d055d
spec:
  deploy:
  - kapp:
      inspect:
        rawOptions:
        - --tree=true
      rawOptions:
      - --wait-timeout=5m
      - --diff-changes=true
  fetch:
  - imgpkgBundle:
      image: ghcr.io/making/rbac-mgmt-bundle:0.0.1
  - git:
      ref: origin/main
      url: https://github.com/making/rbac-mgmt-config-example.git
  serviceAccountName: kapp
  syncPeriod: 1m0s
  template:
  - ytt:
      paths:
      - 0/config
      - 0/schema.yaml
      valuesFrom:
      - path: 1/config/rbac-mgmt-values.yaml
  - kbld:
      paths:
      - '-'
      - 0/.imgpkg/images.yml
status:
  conditions:
  - status: "True"
    type: Reconciling
  consecutiveReconcileSuccesses: 1
  deploy:
    exitCode: 0
    finished: true
    startedAt: "2022-06-09T12:37:39Z"
    stdout: |-
      Target cluster 'https://100.64.0.1:443' (nodes: jaguchi-control-plane-hctz6, 8+)
      @@ create namespace/project-x (v1) cluster @@
            0 + apiVersion: v1
            1 + kind: Namespace
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654778259434510087"
            5 +     kapp.k14s.io/association: v1.f70f8b6fb2967b86812536b39d80020b
            6 +   name: project-x
            7 +
      @@ create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: default @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654778259434510087"
            5 +     kapp.k14s.io/association: v1.c13237b7c67cde32294192f2c1a9d551
            6 +   name: clusterrole-admin-binding
            7 +   namespace: default
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: admin
           12 + subjects:
           13 + - kind: User
           14 +   name: peter@example.com
           15 +
      @@ create rolebinding/clusterrole-edit-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654778259434510087"
            5 +     kapp.k14s.io/association: v1.34e8a6791f6ff4053c4fc3c9080c430c
            6 +   name: clusterrole-edit-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: edit
           12 + subjects:
           13 + - kind: User
           14 +   name: alex@example.com
           15 + - kind: ServiceAccount
           16 +   name: ci-bot
           17 +   namespace: null
           18 +
      @@ create rolebinding/clusterrole-view-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654778259434510087"
            5 +     kapp.k14s.io/association: v1.e317933df089827627ab50148e62f214
            6 +   name: clusterrole-view-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: view
           12 + subjects:
           13 + - kind: User
           14 +   name: oliver@example.com
           15 +
      @@ create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654778259434510087"
            5 +     kapp.k14s.io/association: v1.19b6bf6062f0c08134ca05b009daacb5
            6 +   name: clusterrole-admin-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: admin
           12 + subjects:
           13 + - kind: User
           14 +   name: peter@example.com
           15 +
      @@ create rolebinding/clusterrole-cluster-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x @@
            0 + apiVersion: rbac.authorization.k8s.io/v1
            1 + kind: RoleBinding
            2 + metadata:
            3 +   labels:
            4 +     kapp.k14s.io/app: "1654778259434510087"
            5 +     kapp.k14s.io/association: v1.16d7880c92365169f138c22019960642
            6 +   name: clusterrole-cluster-admin-binding
            7 +   namespace: project-x
            8 + roleRef:
            9 +   apiGroup: rbac.authorization.k8s.io
           10 +   kind: ClusterRole
           11 +   name: cluster-admin
           12 + subjects:
           13 + - kind: Group
           14 +   name: administrators
           15 +
      Changes
      Namespace  Name                               Kind         Conds.  Age  Op      Op st.  Wait to    Rs  Ri
      (cluster)  project-x                          Namespace    -       -    create  -       reconcile  -   -
      default    clusterrole-admin-binding          RoleBinding  -       -    create  -       reconcile  -   -
      project-x  clusterrole-admin-binding          RoleBinding  -       -    create  -       reconcile  -   -
      ^          clusterrole-cluster-admin-binding  RoleBinding  -       -    create  -       reconcile  -   -
      ^          clusterrole-edit-binding           RoleBinding  -       -    create  -       reconcile  -   -
      ^          clusterrole-view-binding           RoleBinding  -       -    create  -       reconcile  -   -
      Op:      6 create, 0 delete, 0 update, 0 noop, 0 exists
      Wait to: 6 reconcile, 0 delete, 0 noop
      12:37:39PM: ---- applying 2 changes [0/6 done] ----
      12:37:39PM: create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: default
      12:37:40PM: create namespace/project-x (v1) cluster
      12:37:40PM: ---- waiting on 2 changes [0/6 done] ----
      12:37:40PM: ok: reconcile namespace/project-x (v1) cluster
      12:37:40PM: ok: reconcile rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: default
      12:37:40PM: ---- applying 4 changes [2/6 done] ----
      12:37:40PM: create rolebinding/clusterrole-cluster-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: create rolebinding/clusterrole-view-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: create rolebinding/clusterrole-edit-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: create rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: ---- waiting on 4 changes [2/6 done] ----
      12:37:40PM: ok: reconcile rolebinding/clusterrole-cluster-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: ok: reconcile rolebinding/clusterrole-edit-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: ok: reconcile rolebinding/clusterrole-admin-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: ok: reconcile rolebinding/clusterrole-view-binding (rbac.authorization.k8s.io/v1) namespace: project-x
      12:37:40PM: ---- applying complete [6/6 done] ----
      12:37:40PM: ---- waiting complete [6/6 done] ----
      Succeeded
    updatedAt: "2022-06-09T12:37:40Z"
  fetch:
    exitCode: 0
    startedAt: "2022-06-09T12:37:33Z"
    stdout: |
      apiVersion: vendir.k14s.io/v1alpha1
      directories:
      - contents:
        - imgpkgBundle:
            image: ghcr.io/making/rbac-mgmt-bundle@sha256:9be53eeaff24345d292f38fc4c5037d8a08496183369b8ac41b36f0569dec8cb
            tag: 0.0.1
          path: .
        path: "0"
      - contents:
        - git:
            commitTitle: Create README.md
            sha: d755004dbc04a89b90b90c2e099cbf5b71ac3fda
          path: .
        path: "1"
      kind: LockConfig
    updatedAt: "2022-06-09T12:37:39Z"
  friendlyDescription: Reconciling
  inspect:
    exitCode: 0
    stdout: |-
      Target cluster 'https://100.64.0.1:443' (nodes: jaguchi-control-plane-hctz6, 8+)
      12:37:40PM: info: Resources: Ignoring group version: schema.GroupVersionResource{Group:"stats.antrea.tanzu.vmware.com", Version:"v1alpha1", Resource:"networkpolicystats"}: feature NetworkPolicyStats disabled
      12:37:40PM: info: Resources: Ignoring group version: schema.GroupVersionResource{Group:"stats.antrea.tanzu.vmware.com", Version:"v1alpha1", Resource:"antreaclusternetworkpolicystats"}: feature NetworkPolicyStats disabled
      12:37:40PM: info: Resources: Ignoring group version: schema.GroupVersionResource{Group:"stats.antrea.tanzu.vmware.com", Version:"v1alpha1", Resource:"antreanetworkpolicystats"}: feature NetworkPolicyStats disabled
      Resources in app 'rbac-mgmt-ctrl'
      Namespace  Name                               Kind         Owner  Conds.  Rs  Ri  Age
      project-x  clusterrole-cluster-admin-binding  RoleBinding  kapp   -       ok  -   2s
      project-x  clusterrole-admin-binding          RoleBinding  kapp   -       ok  -   2s
      project-x  clusterrole-edit-binding           RoleBinding  kapp   -       ok  -   2s
      default    clusterrole-admin-binding          RoleBinding  kapp   -       ok  -   3s
      project-x  clusterrole-view-binding           RoleBinding  kapp   -       ok  -   2s
      (cluster)  project-x                          Namespace    kapp   -       ok  -   3s
      Rs: Reconcile state
      Ri: Reconcile information
      6 resources
      Succeeded
    updatedAt: "2022-06-09T12:37:42Z"
  observedGeneration: 2
  template:
    exitCode: 0
    updatedAt: "2022-06-09T12:37:39Z"

```

Update `rbac-mgmt-values.yaml` on the git repository

## Testing locally

```
ytt -f bundle/config -f bundle/schema.yaml -f example/values.yaml
```

## How to create the carvel package

```
VERSION=0.0.2
kbld -f bundle/config --imgpkg-lock-output bundle/.imgpkg/images.yml
imgpkg push -b ghcr.io/making/rbac-mgmt-bundle:${VERSION} -f bundle
```

```
ytt -f bundle/schema.yaml --data-values-schema-inspect -o openapi-v3 > /tmp/rbac-mgmt-schema-openapi.yml
ytt -f template/package.yaml --data-value-file openapi=/tmp/rbac-mgmt-schema-openapi.yml -v version=${VERSION} > repo/packages/rbac-mgmt.pkg.maki.lol/${VERSION}.yaml

kbld -f repo/packages --imgpkg-lock-output repo/.imgpkg/images.yml
imgpkg push -b ghcr.io/making/rbac-mgmt-repo:${VERSION} -f repo
```