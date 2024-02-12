# Explore Cluster DNS

coredns is running as a deployment in the cluster in ```kube-system``` namespace.

```k -n kube-system get pods -l k8s-app=kube-dns```

```
NAME                       READY   STATUS    RESTARTS        AGE
coredns-5bbc984bcc-x67vm   1/1     Running   0 (3m24s ago)   3m49s
coredns-5bbc984bcc-zkt6q   1/1     Running   0 (3m41s ago)   3m49s
```

Coredns is configured with a ```configmap```

Let's take a look at it.

```k -n kube-system get cm coredns -o yaml```

Output:

```
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2022-12-13T04:55:14Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "117025"
  uid: ed779659-20a2-4f55-b0c5-6006762955d3
```

We will edit the configmap so we can see some logs.

```k -n kube-system edit cm coredns```

```diff
...
data:
  Corefile: |
    .:53 {
+       log
        errors
        health {
           lameduck 5s
        }
...
```

Then we'll spin up some pods and make some queries.

nginx pod:

```k run nginx --image nginx```

expose it:

```k expose pod nginx --port 80```

busybox pod for queries:

```k run busy --image busybox:1.28.0 -- sleep 5000```

Now, open another SSM session so we can have two windows. In one we'll make some queries, in the other we'll tail the logs of coredns.

Follow coredns logs:

```k -n kube-system logs -l k8s-app=kube-dns -f```

Output:

```
.:53
[INFO] plugin/reload: Running configuration MD5 = 3d3f6363f05ccd60e0f885f0eca6c5ff
CoreDNS-1.8.6
linux/amd64, go1.17.1, 13a9191
[INFO] 127.0.0.1:42988 - 29993 "HINFO IN 5480274337352597745.2612353223994470322. udp 57 false 512" NXDOMAIN qr,rd,ra 132 0.095046848s
.:53
[INFO] plugin/reload: Running configuration MD5 = 3d3f6363f05ccd60e0f885f0eca6c5ff
CoreDNS-1.8.6
```

Now: ```k exec busy -c busy -- nslookup nginx```

Output:

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
```

And coredns will log that query:

```
...
[INFO] 192.168.208.199:47773 - 4 "A IN nginx.default.svc.cluster.local. udp 49 false 512" NOERROR qr,aa,rd 96 0.00021347s
[INFO] 192.168.208.199:45642 - 2 "PTR IN 10.0.96.10.in-addr.arpa. udp 41 false 512" NOERROR qr,aa,rd 116 0.000180901s
[INFO] 192.168.208.199:54194 - 5 "PTR IN 228.255.105.10.in-addr.arpa. udp 45 false 512" NOERROR qr,aa,rd 117 0.000143716s
[INFO] 192.168.208.199:43621 - 3 "AAAA IN nginx.default.svc.cluster.local. udp 49 false 512" NOERROR qr,aa,rd 142 0.000203254s
[INFO] 192.168.208.199:33795 - 4 "A IN nginx.default.svc.cluster.local. udp 49 false 512" NOERROR qr,aa,rd 96 0.000185963s
```

Let's look at the resolv.conf for the pod:

```k exec busy -c busy -- cat /etc/resolv.conf```

Output:
```
search default.svc.cluster.local svc.cluster.local cluster.local ap-southeast-2.compute.internal
nameserver 10.96.0.10
options ndots:5
```

10.96.0.10 is the coredns service:

```k -n kube-system get svc```

```
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   18h
```

Take a look at ```/var/lib/kubelet/config.yaml```. Find the configuration for ```clusterDNS```. Kubelet gives the containers DNS resolver information based on this value. It's set to the coredns service IP.

```clusterDomain``` is also configured. This is where the ```cluster.local``` is coming from. 

Let's try an external query:

```k exec busy -c busy -- nslookup google.com```

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      google.com
Address 1: 2404:6800:4006:813::200e syd09s24-in-x0e.1e100.net
Address 2: 172.217.24.46 hkg07s23-in-f14.1e100.net
```
Coredns logs:

This can't be resolved within the cluster, and the query will fallthrough to the specified zones. The fallthrough plugin configuration allows this:

```
...
   fallthrough in-addr.arpa ip6.arpa
...

```

```
[INFO] 192.168.208.199:41384 - 3 "AAAA IN google.com. udp 28 false 512" NOERROR qr,rd,ra 66 0.001585089s
[INFO] 192.168.208.199:32993 - 4 "A IN google.com. udp 28 false 512" NOERROR qr,rd,ra 54 0.003251507s
[INFO] 192.168.208.199:59562 - 2 "PTR IN 10.0.96.10.in-addr.arpa. udp 41 false 512" NOERROR qr,aa,rd 116 0.000259761s
[INFO] 192.168.208.199:34717 - 5 "PTR IN e.0.0.2.0.0.0.0.0.0.0.0.0.0.0.0.3.1.8.0.6.0.0.4.0.0.8.6.4.0.4.2.ip6.arpa. udp 90 false 512" NOERROR qr,rd,ra 201 0.002909143s
[INFO] 192.168.208.199:60520 - 6 "PTR IN 46.24.217.172.in-addr.arpa. udp 44 false 512" NOERROR qr,rd,ra 239 0.00189579s
```

Let's try another internal query, to another namespace.

```k create ns other-ns```

```k -n other-ns run nginx-other --image nginx```

```k -n other-ns expose pod nginx-other --port 80```

Check service looks good (has an endpoint).

```k -n other-ns describe svc nginx-other```

(Following our coredns logs again)

We'll use our busybox pod again:

```k exec busy -c busy -- nslookup nginx-other```

```
nslookup: can't resolve 'nginx-other'
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

command terminated with exit code 1
```

Coredns logs:

```
[INFO] 192.168.208.199:38616 - 3 "AAAA IN nginx-other.default.svc.cluster.local. udp 55 false 512" NXDOMAIN qr,aa,rd 148 0.000282936s
[INFO] 192.168.208.199:41160 - 4 "AAAA IN nginx-other.svc.cluster.local. udp 47 false 512" NXDOMAIN qr,aa,rd 140 0.000943944s
[INFO] 192.168.208.199:56185 - 5 "AAAA IN nginx-other.cluster.local. udp 43 false 512" NXDOMAIN qr,aa,rd 136 0.000179277s
[INFO] 192.168.208.199:55541 - 6 "AAAA IN nginx-other.ap-southeast-2.compute.internal. udp 61 false 512" NXDOMAIN qr,rd,ra 184 0.001408135s
[INFO] 192.168.208.199:57315 - 7 "AAAA IN nginx-other. udp 29 false 512" NXDOMAIN qr,rd,ra 104 0.003251781s
[INFO] 192.168.208.199:43967 - 8 "A IN nginx-other.default.svc.cluster.local. udp 55 false 512" NXDOMAIN qr,aa,rd 148 0.000211542s
[INFO] 192.168.208.199:42152 - 9 "A IN nginx-other.svc.cluster.local. udp 47 false 512" NXDOMAIN qr,aa,rd 140 0.000138975s
[INFO] 192.168.208.199:59662 - 10 "A IN nginx-other.cluster.local. udp 43 false 512" NXDOMAIN qr,aa,rd 136 0.000122309s
[INFO] 192.168.208.199:50165 - 11 "A IN nginx-other.ap-southeast-2.compute.internal. udp 61 false 512" NXDOMAIN qr,rd,ra 184 0.002102657s
[INFO] 192.168.208.199:53388 - 12 "A IN nginx-other. udp 29 false 512" NXDOMAIN qr,rd,ra 104 0.003047049s
```

Notice the query domains. ```nginx-other.default``` 

Default is the namespace. We need to specify ```other-ns```

Read about dns records for pods and services [in k8s docs](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

```k exec busy -c busy -- nslookup nginx-other.other-ns```

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-other.other-ns
Address 1: 10.98.129.168 nginx-other.other-ns.svc.cluster.local
```

success:

coredns logs:
```
[INFO] 192.168.208.199:47605 - 2 "PTR IN 10.0.96.10.in-addr.arpa. udp 41 false 512" NOERROR qr,aa,rd 116 0.000228401s
[INFO] 192.168.208.199:46769 - 3 "AAAA IN nginx-other.other-ns. udp 38 false 512" NXDOMAIN qr,rd,ra 113 0.004010449s
[INFO] 192.168.208.199:38490 - 5 "AAAA IN nginx-other.other-ns.svc.cluster.local. udp 56 false 512" NOERROR qr,aa,rd 149 0.000412764s
[INFO] 192.168.208.199:56706 - 4 "AAAA IN nginx-other.other-ns.default.svc.cluster.local. udp 64 false 512" NXDOMAIN qr,aa,rd 157 0.000274092s
[INFO] 192.168.208.199:60091 - 8 "A IN nginx-other.other-ns.svc.cluster.local. udp 56 false 512" NOERROR qr,aa,rd 110 0.000379928s
[INFO] 192.168.208.199:34993 - 6 "A IN nginx-other.other-ns. udp 38 false 512" NXDOMAIN qr,rd,ra 113 0.002522575s
[INFO] 192.168.208.199:58145 - 7 "A IN nginx-other.other-ns.default.svc.cluster.local. udp 64 false 512" NXDOMAIN qr,aa,rd 157 0.000198867s
[INFO] 192.168.208.199:34049 - 9 "PTR IN 168.129.98.10.in-addr.arpa. udp 44 false 512" NOERROR qr,aa,rd 122 0.001097448s
```
Clean up:

```bash
k delete ns other-ns
k delete pod busy --force
k delete pod nginx
k delete svc nginx
```
