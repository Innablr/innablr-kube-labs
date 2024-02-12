# Assign pods to nodes

The simplest way to assign pods to node is using the labels and selector mechanism. You can label your nodes and then in your pod definition select the label. 

```
$ kubectl get nodes --show-labels
```

Let's label the node, grab the name of the node

```
$ kubectl label nodes <name of the node> type=worker
```

Now let's create the nginx pod with node selector

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    type: worker
```

```
$ kubectl apply -f nginx-test.yaml
```

Verify

```
$ kubectl get pods -o wide
```

You can also schedule pods to specific node using its name

```
...
  spec:
    nodeName: <name of the node>
    containers:
```

## Affinity rules

You can also use node affinity and pod affinity rules for scheduling decisions.

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector