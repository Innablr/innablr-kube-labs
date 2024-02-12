# Where can we access logs for various components and workloads?

Depending on the situation, you may need to leverage different ways to view logs.

## As client we can view container logs with kubectl commands

```k logs <pod-name> -c <container-name>```

- can select by label(s) ```-l k8s-app=kube-dns```
- follow ```-f```
- previous logs```--previous```

## on the host

**log files**


These directories on the node are where kubelet puts container logs. 

```/var/log/pods/``` / ```/var/log/containers```

>Note: that ```/var/log/containers``` is being deprecated at some stage.

**With ```crictl```**

```crictl logs <container-id>```


## kubelet logs

```journalctl -u kubelet```

## calico

- container logs, but if it keeps crashing, could look at log files.

```/var/log/calico/cni/cni.log```

Have a look around.
