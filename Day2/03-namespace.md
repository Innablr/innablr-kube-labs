# Namespace

Namespaces help provide some isolation to certain resources. Let's see which resources are namespaced:

```k api-resources --namespaced=true```

Create namespace using kubectl

```
kubctl create namespace eduk8s
```
or

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: eduk8s
spec: {}
status: {}
```

# Change context to your namespace

```
kubectl config set-context --current --namespace=eduk8s
```


# Exercise
1. Now, deploy your deployment and service to your namespace
2. Delete your namespace.
