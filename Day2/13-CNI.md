# CNI

Where kubelet looks to know what CNI to use:

```ls /etc/cni/net.d/10-calico.conflist```


```
10-calico.conflist  100-crio-bridge.conf  200-loopback.conf  calico-kubeconfig
```