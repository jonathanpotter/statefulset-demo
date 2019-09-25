# Stateful Set Demo

Kubernetes provides two similar controller objects, `Deployment` and `StatefulSet`. Generally, stateless applications should use `Deployment` objects and stateful applications should use `StatefulSet`.

Stateful Sets are useful when the desired number of pod replicas is roughly static and the pods needs a sticky identity. Scaling horizontally with Stateful Sets is not as elegant as with a Deployment, but can be done with more effort. If you find yourself scaling up or down a Stateful Set frequently, it may be a sign that a Deployment is a better fit for your use case.

Here are some exercise to demonstrate the behavior of Stateful Sets. These exercises are optimized to run in CaaS localdev. However, they can be easily modified to run in production CaaS.

## Storage Impacts

When volumes are mounted in pod replicas of a `Deployment`, each replica uses the same `PersistentVolumeClaim` and therefore the same `Volume`. For example, if pod A writes a file to a `VolumeMount` location such as `/mnt/vol1`, then pod B would also be able to access that file at `/mnt/vol1`.

In contrast to a `Deployment`, when volumes are mounted in pod replicas of a `StatefulSet`, each replica gets its own `PersistentVolumeClaim` and unique `Volume`. In this case, a file written to a `VolumeMount` location by pod A, would not be accessible by pod B. The system will also ensure that if a `StatefulSet` pod is restarted, it will get reconnected to the same `Volume` and all of the data will still be intact.

In fact, a `PersistentVolumeClaim` that is created by a `volumeClaimTemplate` of a `StatefulSet`, is really persistent. Even if the `StatefulSet` is deleted by `kubectl delete -f ./statefulset.yaml`, the `PersistentVolumeClaim` is not altered. The PVC can only be deleted explicitly with `kubectl delete pvc MY_PVC`. This will also delete the associated `PersistentVolume` and all data is deleted.

Stateful set pods also keep their same name although their IP addresses will change. This is useful for stateful technologies that write data to a specific location using a DNS name and expect that data to persist at that location.

### Exercise

You'll need a Kubernetes namespace. Create a pull secret in your namespace with access to `registry.ford.com` naming the pull secret the same as in `statefulset.yaml` or modifying the file to match the name of your pull secret.

```
# Create a namespace for objects
oc new-project statefulset

# Check what is in namespace already
kubectl get all       # No stateful sets or pods
kubectl get pvc       # No persistent volume claims
kubectl get pv        # No persistent volumes

# Create the StatefulSet
kubectl create -f ./statefulset.yaml
```

Now you should have a stateful set and two pods. Each pod has its own PVC and each PVC is bound to a unique Persistent Volume. Notice that the pod name is not random, but stable and predictable. Hostnames are created from the name of the StatefulSet and the ordinal of the Pod. The pattern for the constructed hostname is $(statefulset name)-$(ordinal).

```
kubectl get pvc
kubectl get all
```

Write some sample data from one pod to a volume. See that the data is not available from the other pod.

```
oc rsh pod/demo-0
  echo "Hello in demo-0." > /mnt/vol1/aa.txt
  cat /mnt/vol1/aa.txt
  # Hello in demo-0.
  exit

oc rsh pod/demo-1
  cat /mnt/vol1/aa.txt
  # cat: /mnt/vol1/aa.txt: No such file or directory
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

## Routing Impacts

Stateful Sets are also useful when you need a client to be routed to the same pod for each call. Stateful technologies like Casandra, MongoDB, and others require this functionality. Create a headless service to expose the individual pod replicas by name.

### Exercise

Create the service. See that the service contains the current endpoint IP addresses. Check that you can reach a specific pod using a DNS name derived from the pod hostname and subdomain provided by the headless service.

```
kubectl create -f ./service.yaml
kubectl get services
kubectl describe service demo

curl -I demo-0.demo.statefulset.svc.cluster.local:8080/api/v1/hello
```

Delete the pod and it will be restarted with a new IP address, but the DNS name remains the same.

```
kubectl describe pod demo-0 | grep IP:
curl -v demo-0.demo.statefulset.svc.cluster.local:8080/api/v1/hello

kubectl delete pod demo-0
# Wait about 15 seconds for pod restart

kubectl describe pod demo-0 | grep IP:
curl -v demo-0.demo.statefulset.svc.cluster.local:8080/api/v1/hello
```

## Scaling Scenario

You can scale horizontally with Stateful Sets, but note that if PVCs and PVs are created during a scale up event, those objects are persistent after scale down. They must be deleted manually if that is the desired behavior. Additionally, Kubernetes does not notify clients calling a Stateful Set that a scale event has occurred. For these reasons, scaling with a Stateful Set is not as straightforward as scaling with a Deployment fronted by a Load Balancer.

### Exercise

Scale up the Stateful Set from 2 Pods to 3 Pods.

```
kubectl get pods
kubectl get pvc
kubectl scale --replicas=3 statefulset demo

kubectl get pods
kubectl get pvc
```

Note that the new Pod name and new PVC name are predictable and the lists of names are ordered. Write some data to the PV. Scale down the stateful set. Notice that with a Stateful Set, Kubernetes will remove the Pod with the highest ordinal value, Pod `demo-2` in this case. Note that the associated PVC remains.

```
oc rsh pod/demo-2
  echo "Hello in demo-2." > /mnt/vol1/aa.txt
  cat /mnt/vol1/aa.txt
  # Hello in demo-2.
  exit

kubectl scale --replicas=2 statefulset demo

kubectl get pods
kubectl get pvc
```

If you wanted to delete the PVC associated with Pod `demo-2`, you would have to do that manually; otherwise, the data will be there waiting for Pod `demo-2` should it return. Scale up and check that the data persisted.

```
kubectl scale --replicas=3 statefulset demo

oc rsh pod/demo-2
  cat /mnt/vol1/aa.txt
  # Hello in demo-2.
  exit
```
