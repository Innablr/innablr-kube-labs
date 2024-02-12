# Volumes

Create a ```StorageClass```

```bash
k apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

Now let's create a ```local``` ```PersistentVolume```. 

First let's create the directory on the host, on our controlplane node:

```mkdir /mnt/pv-data```

```bash
k apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cp-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/pv-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-role.kubernetes.io/control-plane
          operator: In
          values:
          - "" 
EOF
```

Let's look at the persistent volume

```k get pv```

Output:
```
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
cp-pv   100Gi      RWO            Delete           Available           local-storage            4s
```

It's status is available. Let's use it. We'll run a pod ```cp-pv-user``` and it will make use of a ```PersistentVolumeClaim```

Let's create the ```pvc```

```bash
k apply -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cp-pv-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-storage
EOF
```

```k get pvc```

The status of our claim is pending. our ```local-storage``` storage class has a ```volumeBindingMode``` of ```WaitForFirstConsumer```. The PVC will remain pending until a pod consumes this claim.


Output:
```
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    AGE
cp-pv-pvc   Pending                                      local-storage   3s
```

Create a pod to consume our volume. Note the tolerations.

```bash
k apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: cp-pv-pod
  name: cp-pv-pod
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - echo "I have some volume" > /cp-pv-pod/message.txt;
    image: busybox:1.28.0
    name: cp-pv-pod
    volumeMounts:
    - name: cp-pv
      mountPath: /cp-pv-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Equal"
    value: ""
    effect: "NoSchedule"
  - key: "node-role.kubernetes.io/master"
    operator: "Equal"
    value: ""
    effect: "NoSchedule"
  volumes:
    - name: cp-pv
      persistentVolumeClaim:
        #the claim we made prior
        claimName: cp-pv-pvc
EOF
```

Output:
```
pod/cp-pv-pod configured
```

```k get pods```

```
NAME         READY   STATUS              RESTARTS      AGE
cp-pv-pod    0/1     ContainerCreating   0             51s
```

Check the status of the pvc, it should bind now.


```k get pvc```

```
NAME        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    AGE
cp-pv-pvc   Bound    cp-pv    100Gi      RWO            local-storage   5m
```


On our host we can confirm the volume was sucesfully utilized:

cat /mnt/pv-data/message.txt

```
I have some volume
```

we can now cleanup:

```k delete pod cp-pv-pod```

```k delete pvc cp-pv-pvc```

```k delete pv cp-pv```

---