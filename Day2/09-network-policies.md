# Network Policies

```
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```

```
$ kubectl create deployment nginx --image=nginx
```

```
$ kubectl expose deployment nginx --port=80
```


```
$ kubectl run busybox --rm -ti --image=busybox:1.28 -- /bin/sh
$ wget --spider --timeout=1 nginx
```


```
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
```

```
$ kubectl apply -f https://k8s.io/examples/service/networking/nginx-policy.yaml
```

```
$ kubectl run busybox --rm -ti --image=busybox:1.28 -- /bin/sh
$ wget --spider --timeout=1 nginx
```

```
$ kubectl run busybox --rm -ti --labels="access=true" --image=busybox:1.28 -- /bin/sh
wget --spider --timeout=1 nginx
```
