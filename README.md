# DESC\_Kubernates\_PVC\_Demo

A basic Kubernetes/K3s demo for understanding how to mount a **PersistentVolumeClaim (PVC)** inside a Pod and then move from a standalone Pod to a **Deployment-managed Pod** as a better practice.

This repository demonstrates the following Kubernetes storage flow:

```text
StorageClass → PersistentVolume → PersistentVolumeClaim → Pod/Deployment → Container mountPath
```

\---

\---

## Demo Goal

This demo shows how persistent storage works in Kubernetes/K3s using an Nginx container.

The PVC is mounted inside the container at:

```text
/usr/share/nginx/html
```

A custom `index.html` file is written inside this mounted path.

The demo first uses a standalone Pod so the PVC mounting concept is easy to understand.

Then the standalone Pod is deleted and a Deployment is applied to recreate and manage the Pod.

This shows two important points:

```text
1. Data persists even after the Pod is deleted.
2. A Deployment is the better way to run application Pods.
```

\---

## Demo Overview

|Demo Item|Value|
|-|-|
|Cluster type|K3s single-node or multi-node cluster|
|StorageClass|`local-path`|
|Namespace|`storage-demo`|
|PVC name|`web-pvc`|
|Initial Pod name|`nginx-pvc-pod`|
|Deployment name|`nginx-pvc-deployment`|
|Service name|`nginx-pvc-service`|
|Container image|`nginx:alpine`|
|PVC mount path|`/usr/share/nginx/html`|
|Storage requested|`1Gi`|

\---

## Files in This Repository

```text
DESC\_Kubernates\_PVC\_Demo/
├── 01-namespace.yaml
├── 02-pvc.yaml
├── 03-pod.yaml
├── 04-service.yaml
└── 05-deployment.yaml
```

|File|Purpose|
|-|-|
|`01-namespace.yaml`|Creates a separate namespace for the demo|
|`02-pvc.yaml`|Creates the PersistentVolumeClaim that requests `1Gi` storage|
|`03-pod.yaml`|Creates a standalone Nginx Pod and mounts the PVC into the container|
|`04-service.yaml`|Creates a Service to expose the Nginx application inside the cluster|
|`05-deployment.yaml`|Creates a Deployment that manages an Nginx Pod using the same PVC|

> If your repository file is named `05-deployment-optional.yaml`, you can either rename it to `05-deployment.yaml` or replace the command in this README with your actual file name.

\---

## Concept Flow

The core storage flow is:

```text
StorageClass
   ↓
PersistentVolume
   ↓
PersistentVolumeClaim
   ↓
Pod or Deployment-created Pod
   ↓
Container mountPath
```

For this demo:

```text
local-path StorageClass
   ↓
PV is created automatically
   ↓
PVC requests 1Gi storage
   ↓
Nginx Pod mounts PVC
   ↓
Data is written to /usr/share/nginx/html
   ↓
Standalone Pod is deleted
   ↓
Deployment recreates an Nginx Pod using the same PVC
   ↓
Data is still available
```

\---

## Key Concepts

|Concept|Meaning|
|-|-|
|StorageClass|A profile/type of storage. In K3s, `local-path` is usually available by default|
|PersistentVolume, PV|The actual storage resource in the cluster|
|PersistentVolumeClaim, PVC|A request for storage. The Pod uses the PVC, not the PV directly|
|volumeMount|Defines where a volume is mounted inside a container|
|mountPath|The exact container path where the volume appears|
|Pod|The smallest deployable unit in Kubernetes|
|Deployment|A controller that creates and manages Pods|
|Reclaim Policy|Defines what happens to the PV after the PVC is deleted|

\---

## Pod vs Deployment

### What is a Pod?

A **Pod** is the smallest object that Kubernetes runs.

A Pod contains one or more containers.

In this demo, the standalone Pod is used first because it makes the PVC mounting concept simple:

```text
Pod → Container → PVC mounted at /usr/share/nginx/html
```

However, a standalone Pod is not ideal for real applications.

If a standalone Pod is deleted, Kubernetes does not automatically recreate it unless another controller manages it.

\---

### What is a Deployment?

A **Deployment** is a higher-level Kubernetes object that manages Pods.

A Deployment creates Pods using a Pod template.

If a Deployment-managed Pod is deleted, the Deployment automatically creates a replacement Pod.

Deployment flow:

```text
Deployment → ReplicaSet → Pod → Container
```

\---

## How Pod and Deployment Are Similar

A standalone Pod and a Deployment-created Pod can run the same container and mount the same PVC.

Both use similar Pod-level configuration:

```text
containers
volumeMounts
volumes
persistentVolumeClaim
mountPath
```

Example:

```text
Standalone Pod:
Pod directly contains container + volumeMount + PVC reference

Deployment:
Deployment contains a Pod template
The Pod template contains container + volumeMount + PVC reference
```

So the actual mounting logic is the same.

\---

## How Pod and Deployment Are Different

|Area|Standalone Pod|Deployment|
|-|-|-|
|Purpose|Basic/manual workload|Application workload management|
|Self-healing|No automatic recreation|Automatically recreates failed/deleted Pods|
|Scaling|Not suitable|Supports replicas|
|Rolling updates|Not supported directly|Supports rolling updates and rollbacks|
|Best use|Testing, debugging, simple demos|Real applications and production workloads|
|YAML location of container spec|Directly under `spec`|Under `spec.template.spec`|
|Pod name|Fixed name|Generated name based on Deployment/ReplicaSet|

\---

## Best Practice

Use a standalone Pod for:

```text
- quick testing
- debugging
- simple demonstrations
- learning basic Pod structure
```

Use a Deployment for:

```text
- running applications
- self-healing workloads
- rolling updates
- rollback support
- replica management
- production-style deployments
```

For real application workloads, prefer:

```text
Deployment + Service + PVC
```

instead of:

```text
Standalone Pod + Service + PVC
```

\---

## Important Storage Note

This demo uses:

```yaml
accessModes:
  - ReadWriteOnce
```

`ReadWriteOnce`, or `RWO`, means the volume can be mounted as read-write by one node at a time.

For this reason, the Deployment should use:

```yaml
replicas: 1
```

Do not increase replicas in this demo unless your storage backend supports the required access mode.

For shared multi-replica access, a storage backend supporting `ReadWriteMany`, or `RWX`, is usually required.

\---

## Prerequisites

Before running this demo, make sure:

* A K3s or Kubernetes cluster is running
* `kubectl` is installed and configured
* The current user has permission to create namespaces, PVCs, Pods, Services, and Deployments
* In K3s, the `local-path` StorageClass is available

Check cluster nodes:

```bash
kubectl get nodes
```

Expected example:

```text
NAME      STATUS   ROLES                  AGE   VERSION
master    Ready    control-plane,master   10m   v1.xx.x+k3s1
```

\---

## Check Available StorageClass

```bash
kubectl get storageclass
```

Expected example in K3s:

```text
NAME                   PROVISIONER
local-path (default)   rancher.io/local-path
```

### Meaning

|Output Item|Meaning|
|-|-|
|`local-path`|Default K3s local storage class|
|`rancher.io/local-path`|Provisioner that creates local persistent volumes|

If `local-path` is missing, the PVC may stay in `Pending` state.

\---

## Step 1: Apply Namespace

```bash
kubectl apply -f 01-namespace.yaml
```

Creates the `storage-demo` namespace.

Verify:

```bash
kubectl get ns
```

Expected result:

```text
storage-demo   Active
```

\---

## Step 2: Apply PVC

```bash
kubectl apply -f 02-pvc.yaml
```

Creates the PVC named `web-pvc`.

Verify:

```bash
kubectl get pvc -n storage-demo
```

Possible output:

```text
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
web-pvc   Pending                                      local-path
```

The PVC may initially show `Pending`.

This is acceptable in K3s because the `local-path` provisioner may wait until a Pod uses the PVC before binding storage.

\---

## Step 3: Apply Standalone Pod

```bash
kubectl apply -f 03-pod.yaml
```

Creates the standalone Nginx Pod and mounts the PVC at:

```text
/usr/share/nginx/html
```

Verify Pod status:

```bash
kubectl get pods -n storage-demo
```

Expected output:

```text
NAME            READY   STATUS    RESTARTS   AGE
nginx-pvc-pod   1/1     Running   0          20s
```

\---

## Step 4: Verify PVC Binding

```bash
kubectl get pvc -n storage-demo
```

Expected output after the Pod starts:

```text
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
web-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx     1Gi        RWO            local-path
```

Check the created PersistentVolume:

```bash
kubectl get pv
```

Expected output:

```text
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            Delete           Bound    storage-demo/web-pvc
```

### Column Meaning

|Column|Meaning|
|-|-|
|`CAPACITY`|Size of the persistent storage|
|`ACCESS MODES`|`RWO` means ReadWriteOnce|
|`RECLAIM POLICY`|Defines what happens when the PVC is deleted|
|`STATUS`|`Bound` means the PV is connected to the PVC|
|`CLAIM`|Shows which PVC is using the PV|

\---

## Step 5: Write Data into the PVC

Open a shell inside the Nginx container:

```bash
kubectl exec -it nginx-pvc-pod -n storage-demo -- sh
```

### Command Meaning

|Command Part|Meaning|
|-|-|
|`kubectl exec`|Runs a command inside a container|
|`-it`|Opens an interactive terminal|
|`nginx-pvc-pod`|Target Pod name|
|`-n storage-demo`|Target namespace|
|`-- sh`|Starts a shell inside the container|

Inside the container, create an `index.html` file:

```bash
echo "Hello from PVC storage in K3s" > /usr/share/nginx/html/index.html
```

Verify the file:

```bash
cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello from PVC storage in K3s
```

Exit the container:

```bash
exit
```

### Key Point

The file is created inside the PVC-mounted directory:

```text
/usr/share/nginx/html
```

This means the file is stored in persistent storage, not only inside the container writable layer.

\---

## Step 6: Apply Service

```bash
kubectl apply -f 04-service.yaml
```

Creates a `ClusterIP` Service for the Nginx application.

Verify:

```bash
kubectl get svc -n storage-demo
```

Expected output:

```text
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
nginx-pvc-service   ClusterIP   10.x.x.x        <none>        80/TCP
```

\---

## Step 7: Access the Nginx Page

Forward local port `8080` to the Service port `80`:

```bash
kubectl port-forward svc/nginx-pvc-service 8080:80 -n storage-demo
```

### Command Meaning

|Part|Meaning|
|-|-|
|`port-forward`|Temporarily forwards traffic|
|`svc/nginx-pvc-service`|Target Service|
|`8080:80`|Local port `8080` maps to Service port `80`|
|`-n storage-demo`|Namespace|

Keep this terminal open.

Open another terminal and run:

```bash
curl http://localhost:8080
```

Expected output:

```text
Hello from PVC storage in K3s
```

\---

## Step 8: Delete the Standalone Pod

Delete only the standalone Pod:

```bash
kubectl delete pod nginx-pvc-pod -n storage-demo
```

Verify the Pod is deleted:

```bash
kubectl get pods -n storage-demo
```

Verify the PVC still exists:

```bash
kubectl get pvc -n storage-demo
```

Expected result:

```text
The Pod is deleted, but the PVC still exists.
```

This proves that the PVC lifecycle is independent of the Pod lifecycle.

\---

## Step 9: Redeploy Using Deployment

Now apply the Deployment manifest.

```bash
kubectl apply -f 05-deployment.yaml
```

If your file is named `05-deployment-optional.yaml`, use:

```bash
kubectl apply -f 05-deployment-optional.yaml
```

### Meaning

This creates a Deployment that manages an Nginx Pod using the same PVC.

Check the Deployment:

```bash
kubectl get deployments -n storage-demo
```

Expected output:

```text
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
nginx-pvc-deployment   1/1     1            1           20s
```

Check the new Pod:

```bash
kubectl get pods -n storage-demo
```

Expected output example:

```text
NAME                                    READY   STATUS    RESTARTS   AGE
nginx-pvc-deployment-xxxxxxxxx-xxxxx    1/1     Running   0          20s
```

Notice that the Pod name is generated by the Deployment.

\---

## Step 10: Verify Data Persistence from Deployment Pod

Get the Deployment-created Pod name:

```bash
kubectl get pods -n storage-demo
```

Then check the file:

```bash
kubectl exec -it <DEPLOYMENT-POD-NAME> -n storage-demo -- cat /usr/share/nginx/html/index.html
```

Example:

```bash
kubectl exec -it nginx-pvc-deployment-xxxxxxxxx-xxxxx -n storage-demo -- cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello from PVC storage in K3s
```

### Result

The data was created by the first standalone Pod.

Then the standalone Pod was deleted.

A Deployment created a new Pod.

The new Pod mounted the same PVC and could still read the file.

This proves that:

```text
PVC data survives Pod deletion.
Deployment is a better way to run and manage application Pods.
```

\---

## Step 11: Test Deployment Self-Healing

Delete the Deployment-created Pod:

```bash
kubectl delete pod <DEPLOYMENT-POD-NAME> -n storage-demo
```

Check Pods again:

```bash
kubectl get pods -n storage-demo
```

Expected result:

```text
A new Pod is automatically created by the Deployment.
```

Verify the file again:

```bash
kubectl get pods -n storage-demo
kubectl exec -it <NEW-DEPLOYMENT-POD-NAME> -n storage-demo -- cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello from PVC storage in K3s
```

### Key Point

A Deployment keeps the desired state.

If the Deployment says one replica should run, Kubernetes will keep one Pod running.

\---

## Full Command Sequence

```bash
# Pre-checks
kubectl get nodes
kubectl get storageclass

# Apply namespace
kubectl apply -f 01-namespace.yaml
kubectl get ns

# Apply PVC
kubectl apply -f 02-pvc.yaml
kubectl get pvc -n storage-demo

# Apply standalone Pod
kubectl apply -f 03-pod.yaml
kubectl get pods -n storage-demo

# Verify PVC and PV
kubectl get pvc -n storage-demo
kubectl get pv

# Write file inside PVC-mounted path
kubectl exec -it nginx-pvc-pod -n storage-demo -- sh
echo "Hello from PVC storage in K3s" > /usr/share/nginx/html/index.html
cat /usr/share/nginx/html/index.html
exit

# Apply Service
kubectl apply -f 04-service.yaml
kubectl get svc -n storage-demo

# Access Nginx using port-forward
kubectl port-forward svc/nginx-pvc-service 8080:80 -n storage-demo
```

In another terminal:

```bash
curl http://localhost:8080
```

Delete standalone Pod and redeploy using Deployment:

```bash
# Delete standalone Pod
kubectl delete pod nginx-pvc-pod -n storage-demo

# Confirm PVC still exists
kubectl get pvc -n storage-demo

# Apply Deployment
kubectl apply -f 05-deployment.yaml

# If your file has the old name, use this instead:
# kubectl apply -f 05-deployment-optional.yaml

# Check Deployment and Pods
kubectl get deployments -n storage-demo
kubectl get pods -n storage-demo

# Verify data from the Deployment-created Pod
kubectl exec -it <DEPLOYMENT-POD-NAME> -n storage-demo -- cat /usr/share/nginx/html/index.html
```

Test Deployment self-healing:

```bash
# Delete the Deployment-created Pod
kubectl delete pod <DEPLOYMENT-POD-NAME> -n storage-demo

# Check that Deployment creates a new Pod
kubectl get pods -n storage-demo

# Verify data again from the new Pod
kubectl exec -it <NEW-DEPLOYMENT-POD-NAME> -n storage-demo -- cat /usr/share/nginx/html/index.html
```

\---

## Troubleshooting

|Problem|Command to Check|Likely Cause / Fix|
|-|-|-|
|PVC stays `Pending`|`kubectl get storageclass`|`local-path` StorageClass may be missing or not default|
|PVC stays `Pending`|`kubectl get pods -n kube-system \| grep local-path`|local-path provisioner may not be running|
|Standalone Pod stays `Pending`|`kubectl describe pod nginx-pvc-pod -n storage-demo`|PVC not found, StorageClass issue, or scheduling issue|
|Deployment Pod stays `Pending`|`kubectl describe pod <DEPLOYMENT-POD-NAME> -n storage-demo`|PVC issue, scheduling issue, or access mode issue|
|Service does not respond|`kubectl get pods -n storage-demo --show-labels`|Service selector may not match Pod labels|
|File does not persist|`kubectl exec -it <POD-NAME> -n storage-demo -- ls -l /usr/share/nginx/html`|File may have been written outside the mounted path|
|Deployment creates Pod with different name|`kubectl get pods -n storage-demo`|This is normal; Deployment-generated Pods have generated names|

\---

## Cleanup

Delete the namespace:

```bash
kubectl delete namespace storage-demo
```

This removes:

* Standalone Pod, if still present
* Deployment
* Deployment-created Pods
* PVC
* Service

Check PV status:

```bash
kubectl get pv
```

Depending on the reclaim policy, the PV may be deleted automatically or remain temporarily.

\---

## What Changes in Standard Kubernetes?

The YAML structure remains mostly the same.

The main field that may change is:

```yaml
storageClassName: local-path
```

|Environment|Storage Difference|
|-|-|
|K3s|Usually has `local-path` StorageClass by default|
|Minikube|Usually has `standard` StorageClass|
|kubeadm bare metal|Often has no default StorageClass; may require Longhorn, NFS provisioner, OpenEBS, Ceph CSI, or local-path|
|Cloud Kubernetes|Uses cloud storage classes such as AWS EBS CSI, Azure Disk, or Google Persistent Disk|

\---

## Final Summary

Containers and Pods are temporary.

If data is stored only inside a container, it can disappear when the Pod is deleted.

A PVC allows the Pod to request persistent storage.

The Pod mounts the PVC into a directory such as:

```text
/usr/share/nginx/html
```

Anything written inside that mounted directory is stored in the PersistentVolume, not only inside the container.

This demo proves persistence by:

```text
1. Creating a file inside the PVC-mounted path using a standalone Pod
2. Deleting the standalone Pod
3. Applying a Deployment to create a new Pod
4. Verifying that the new Deployment-created Pod can still read the file
5. Deleting the Deployment-created Pod and observing that the Deployment recreates it
```

Best practice:

```text
Use standalone Pods for learning/testing.
Use Deployments for application workloads.
Use PVCs when data must survive Pod deletion.
```
