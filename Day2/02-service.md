# Services

Deploy the pod using the manifet below

```yaml
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
kubectl expose pod sampleapp2 --port=5000 --target-port=80 --name sampleapp2-service
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
kubectl expose pod sampleapp2 --port=5000 --target-port=80 --name sampleapp2-service
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

Describe the service to see our endpoint. We can see one endpoint (192.168.229.8:80, the pod ip mapped to the ```targetPort``` we specified. 

```k describe svc sampleapp2-service```

Output:
```
Name:              sampleapp2-service
Namespace:         default
Labels:            app=test-app
Annotations:       <none>
Selector:          app=test-app
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.109.63.90
IPs:               10.109.63.90
Port:              <unset>  5000/TCP
TargetPort:        80/TCP
Endpoints:         192.168.229.8:80
Session Affinity:  None
Events:            <none>
```

This service exposes the pods it selects within the cluster. Let's test this out.

Bring up a pod to run some ad hoc commands:

```k run busy --image busybox:1.28.0 -- sleep 5000```

Run an exec command:

This command will execute a command, ```nslookup <args>``` on the ```busy``` pod's ```busy``` container:

```k exec busy -c busy -- nslookup sampleapp2-service```


Output:

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      sampleapp2-service
Address 1: 10.96.20.45 sampleapp2-service.default.svc.cluster.local
```

Let's try hit it:

```k exec busy -c busy -- wget sampleapp2-service```


```
Connecting to sampleapp2-service (10.99.132.23:80)
```

hmmm it's hanging for ages. Let's think.

We know the pod is ready, but let's confirm it is healthy. Let's try to hit directly and bypass the service.

Get the pod IP: ```k get pod sampleapp2 -o wide```

Output:

```
NAME         READY   STATUS    RESTARTS   AGE   IP                NODE                                            NOMINATED NODE   READINESS GATES
sampleapp2   1/1     Running   0          35m   192.168.208.196   ip-10-0-0-196.ap-southeast-2.compute.internal   <none>           <none>
```

And we know our application serves on port 5000, try hit it:

```k exec busy -c busy -- wget 192.168.208.196:5000```

output:

```
Connecting to 192.168.208.196:5000 (192.168.208.196:5000)
index.html           100% |*******************************|    98   0:00:00 ETA
```

That works.

We've confirmed our pod is healthy. Our service is the problem. 

Our service is reachable and has healthy endpoints, we can see them when we describe the service and a nslookup is successful

Let's check the service ports.

```
  ports:
  - port: 5000
    protocol: TCP
    targetPort: 80
```

They are the wrong way around. Our application listens on ```5000```. 

Delete the service ```k delete svc sampleapp2-service```

And recreate it, swapping our port and targetPort:

```kubectl expose pod sampleapp2 --port=80 --target-port=5000 --name sampleapp2-service```

Try hit it again:

```k exec busy -c busy -- wget sampleapp2-service```

Success. We hit it.
```
Connecting to sampleapp2-service (10.104.143.41:80)
wget: can't open 'index.html': File exists
command terminated with exit code 1
```
