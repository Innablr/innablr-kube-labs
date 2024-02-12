# Daemonset

Daemonsets are similar to deployments in that they are a set of pods. Rather than specifying replicas, a daemonset runs one copy of the pod on every node.

We can't create a daemonset with an imperative command, but the spec is similar to that of a deployment. We can create a deployment manifest and make smal edits to get a daemonset manifest.

```k create deployment my-ds --image nginx --dry-run=client -o yaml > ds.yaml```

we remove ```spec.replicas```, remove ```spec.strategy```, and change ```kind``` to ```DaemonSet```. If we want this pod to run on our control-plane nodes, we will have to tolerate it's taints as well.

(excuse the indentation on the diff)

```diff
apiVersion: apps/v1
#change kind
-kind: Deployment
+kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: my-ds
  name: my-ds
spec:
#remove replicas
- replicas: 1
  selector:
    matchLabels:
      app: my-ds
  #remove strategy
- strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: my-ds
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
-status: {}
```

Exercise:

- inspect the control plane node and find what taints it has
- add these tolerations to our DS pods

When we upgrade our manifest we should see the pod running on both nodes.
