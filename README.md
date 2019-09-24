# Stateful Set Demo

Kubernetes provides two similar controller objects, `Deployment` and `StatefulSet`. Generally, stateless applications should use `Deployment` objects and stateful applications should use `StatefulSet`.

# Storage Impacts

When volumes are mounted in pod replicas of a `Deployment`, each replica uses the same `PersistentVolumeClaim` and therefore the same `Volume`. For example, if pod A writes a file to a `VolumeMount` location such as `/mnt/vol1`, then pod B would also be able to access that file at `/mnt/vol1`.

In contrast to a `Deployment`, when volumes are mounted in pod replicas of a `StatefulSet`, each replica gets its own `PersistentVolumeClaim` and unique `Volume`. In this case, a file written to a `VolumeMount` location by pod A, would not be accessible by pod B. The system will also ensure that if a `StatefulSet` pod is restarted, it will get reconnected to the same `Volume` and all of the data will still be intact.

In fact, a `PersistentVolumeClaim` that is created by a `volumeClaimTemplate` of a `StatefulSet`, is really persistent. Even if the `StatefulSet` is deleted by `kubectl delete -f ./statefulset.yaml`, the `PersistentVolumeClaim` is not altered. The PVC can only be deleted explicitly with `kubectl delete pvc MY_PVC`. This will also delete the associated `PersistentVolume` and all data is deleted.

Stateful set pods also keep their same name although their IP addresses will change. This is useful for stateful technologies that write data to a specific node name and expect that data to persist at that location.

# Exercise

You'll need a Kubernetes namespace. Create a pull secret in your namespace with access to `registry.ford.com` naming the pull secret the same as in `statefulset.yaml` or modifying the file to match the name of your pull secret.

```
# Check what is in namespace already
kubectl get all       # No stateful sets or pods
kubectl get pvc       # No persistent volume claims
kubectl get pv        # No persistent volumes

# Create the StatefulSet
kubectl create -f ./statefulset.yaml
```

Now you should have a stateful set and two pods. Each pod has its own PVC and each PVC is bound to a unique Persistent Volume.

```
kubectl get pvc
kubectl get all
```

Write some sample data from one pod to a volume. See that the data is not available from the other pod.

```
oc rsh pod/demo-0
  echo "Hello in demo-0." > /mnt/vol1/aa.txt
  cat /mnt/vol1/aa.txt
  Hello in demo-0.
  exit

oc rsh pod/demo-1
  cat /mnt/vol1/aa.txt
  cat: /mnt/vol1/aa.txt: No such file or directory
  exit  
```

Delete `demo-0` pod. See that the pod is restarted with the same name although the IP address may change. See that the data that was written to the persistent volume is still there.

```
kubectl get pods
kubectl describe pod demo-0 | grep IP:

kubectl delete pod demo-0
# Wait about 15 seconds for pod restart

kubectl get pods
kubectl describe pod demo-0 | grep IP:

oc rsh pod/demo-0
  cat /mnt/vol1/aa.txt
  Hello in demo-0.
  exit
```

# Routing Impacts

Stateful Sets are also useful when you need a client to be routed to the same node for each call. Stateful technologies like Casandra, MongoDB, and others require this functionality.