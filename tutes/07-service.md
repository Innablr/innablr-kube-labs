# Services

Deploy the pod using the manifet below

```
---
apiVersion: v1
kind: Pod
metadata:
  name: sampleapp2
spec:
  containers:
  - name: sampleapp
    image: public.ecr.aws/p9b0l7y4/eduk8s-sampleapp
    env:
    - name: MESSAGE
      value: Personalised greeting from manifest
```

# Expose the pod via kubectl

```
kubectl expose pod sampleapp2 --port=5000 --target-port=80 --name sampleapp-service
```

Uh oh! that would not have worked

Lets look at the error message

```
error: couldn't retrieve selectors via --selector flag or introspection: the pod has no labels and cannot be exposed
```

Now look at the pod spec and see if you can spot the issue

# Recreate Pod with label

Delete the pod

```
kubectl delete pod sampleapp2
```
Now create the pod with labels added to the definition

```YAML
---
apiVersion: v1
kind: Pod
metadata:
  name: sampleapp2
  labels:
    app: test-app
spec:
  containers:
  - name: sampleapp
    image: public.ecr.aws/p9b0l7y4/eduk8s-sampleapp
    env:
    - name: MESSAGE
      value: Personalised greeting from manifest
```

Run the kubectl expose command again

```
kubectl expose pod sampleapp2 --port=5000 --target-port=80 --name sampleapp-service
service/sampleapp2-service exposed
```
# Inspect the service

```
kubectl get service

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes           ClusterIP   10.96.0.1      <none>        443/TCP    9h
sampleapp2-service   ClusterIP   10.96.34.217   <none>        5000/TCP   3s
```

```
kubectl get svc sampleapp2-service -o yaml
```