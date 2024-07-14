
# Set-up MultiKueue environment Tutorial

We use the [Open-Cluster-Manangement environment](https://open-cluster-management.io) as an example to set up a [MultiKueue environment](https://kueue.sigs.k8s.io/docs/concepts/multikueue/). 

For detailed commands and configuration examples, visit the [MultiKueue setup documentation](https://kueue.sigs.k8s.io/docs/tasks/manage/setup_multikueue/).



## Installation

### 1. Worker Cluster Setup:
If you don't have an OCM environment, follow the [Quick-start-Installation](https://open-cluster-management.io/getting-started/quick-start/), to have a `kind-hub` as a manager cluster and two managed clusters `kind-cluster1` and `kind-cluster2`.

- Configure `kubectl` to use the worker cluster. Here we use `kind-cluster3` as a worker cluster.
```bash
kubectl config use kind-cluster3 
```

- Install [Kueue CRDs](https://github.com/kubernetes-sigs/kueue): Ensure that Kueue CRDs are installed in your worker cluster. You can install them using the following command:
```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.7.1/manifests.yaml
```

- Apply the Manifest
```bash
kubectl apply -f single-clusterqueue-setup.yaml
```
You should see:
```bash
resourceflavor.kueue.x-k8s.io/default-flavor created
clusterqueue.kueue.x-k8s.io/cluster-queue created
localqueue.kueue.x-k8s.io/user-queue created
```

### 2. Apply MultiKueue Specific Kubeconfig:
Your current context is still worker cluster.
- Create a restricted MultiKueue role, service account, and role binding. For example: 

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${MULTIKUEUE_SA}
  namespace: ${NAMESPACE}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ${MULTIKUEUE_SA}-role
rules:
- apiGroups:
  - batch
  resources:
  - jobs
  verbs:
  - create
  - delete
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - jobs/status
  verbs:
  - get
- apiGroups:
  - jobset.x-k8s.io
  resources:
  - jobsets
  verbs:
  - create
  - delete
  - get
  - list
  - watch
- apiGroups:
  - jobset.x-k8s.io
  resources:
  - jobsets/status
  verbs:
  - get
- apiGroups:
  - kueue.x-k8s.io
  resources:
  - workloads
  verbs:
  - create
  - delete
  - get
  - list
  - watch
- apiGroups:
  - kueue.x-k8s.io
  resources:
  - workloads/status
  verbs:
  - get
  - patch
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ${MULTIKUEUE_SA}-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ${MULTIKUEUE_SA}-role
subjects:
- kind: ServiceAccount
  name: ${MULTIKUEUE_SA}
  namespace: ${NAMESPACE}
```

- Generate a kubeconfig file for the worker cluster.
```bash
chmod +x create-multikueue-kubeconfig.sh
./create-multikueue-kubeconfig.sh kind-cluster3.kubeconfig
```

And then you will see the restricted MultiKueue role, service account, role binding and secret being created.

```bash
serviceaccount/multikueue-sa created
clusterrole.rbac.authorization.k8s.io/multikueue-sa-role created
clusterrolebinding.rbac.authorization.k8s.io/multikueue-sa-crb created
secret/multikueue-sa created
Writing kubeconfig in kind-cluster3.kubeconfig
```

### 3. Manager Cluster Setup:
- Configure `kubectl` to use the manager cluster.

```bash
kubectl config use kind-hub
```
- Install [Kueue CRDs](https://github.com/kubernetes-sigs/kueue): Ensure that Kueue CRDs are installed in your manager cluster. You can install them using the following command:
```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.7.1/manifests.yaml
```

- Install JobSet on the management cluster. (If using an older version of Kueue than 0.7.0, only install the JobSet CRD in the management cluster.)

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/jobset/releases/download/v0.5.2/manifests.yaml
```

- Enable the MultiKueue Feature
Edit the kueue-controller-manager deployment to enable the MultiKueue feature: 
```bash
kubectl edit deployment kueue-controller-manager -n kueue-system
```

Locate the `containers` section and add the `--feature-gates=MultiKueue=true` argument to the manager container. Your deployment should look similar to this:
```yaml
kind: Deployment
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: manager
        args:
        - --config=/controller_manager_config.yaml
        - --zap-log-level=2
        - --feature-gates=MultiKueue=true  # Add this line
```

- Create a secret with the worker's kubeconfig.
```bash
kubectl create secret generic worker1-secret -n kueue-system --from-file=kubeconfig=kind-cluster3.kubeconfig
```
You can see the secret under the `kueue-system` namespace.
```bash
kubectl get secret worker1-secret -n kueue-system
NAME             TYPE     DATA   AGE
worker1-secret   Opaque   1      22m
```

### 4. Sample Setup:
- Apply a sample configuration manifest for the MultiKueue setup.
```bash
kubectl apply -f multikueue-setup.yaml 
```

- Verify the setup by checking the status of the ClusterQueue, AdmissionCheck, and MultiKueueCluster.
```bash
kubectl get clusterqueues cluster-queue -o jsonpath="{range .status.conditions[?(@.type == \"Active\")]}CQ - Active: {@.status} Reason: {@.reason} Message: {@.message}{'\n'}{end}"
kubectl get admissionchecks sample-multikueue -o jsonpath="{range .status.conditions[?(@.type == \"Active\")]}AC - Active: {@.status} Reason: {@.reason} Message: {@.message}{'\n'}{end}"
kubectl get multikueuecluster kind-cluster3 -o jsonpath="{range .status.conditions[?(@.type == \"Active\")]}MC - Active: {@.status} Reason: {@.reason} Message: {@.message}{'\n'}{end}"

```
