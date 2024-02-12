# RBAC

## auth can-i

```
kubectl auth can-i get pods
kubectl auth can-i get pods --as eduk8s
```

# Create RBAC

Let's create a pod reader role that allows you to get,watch,list
```
kubectl create role -h
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods
```

Let's create a rolebinding

```
kubectl create rolebinding -h
kubectl create rolebinding eduk8s-pod-reader-binding --role=pod-reader --user=eduk8s
```

# Test

```
kubectl auth can-i get pods --as eduk8s
kubectl auth can-i "*" pods --as eduk8s
```

# Create service account

```
kubectl create sa -h
kubectl create sa eduk8s1
```

# Advanced Tip

If you want to check what RBAC change will be applied by your RBAC file, 
```
kubectl auth reconcile -f my-rbac-rules.yaml --dry-run=client
```

# Question 

1. Create a pod-reader role that allows you to get pods in all namespace and assign it to user=innablr.
2. Create a service account eduk8s2 and bind it to the role in 1.



Hint:
kubectl create clusterrolebinding reader-pod-admin- \
  --clusterrole=<cluster-role_name>  \
  --serviceaccount=<namespace>:<serviceAccount_name>
