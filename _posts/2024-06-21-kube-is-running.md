---
layout: post
title: Finally, my first (working) cluster!
image: "/files/rpi-cluster.jpg"
---

![](/files/rpi-cluster.jpg)

After four (or more) years of trying and failing, I finally have a fully working
plain Kubernetes system running on my Raspberry Pi cluster with `kubeadm`.
Persistent storage hasn't been set up yet but `LoadBalancer` services are
available with Cilium and backed with BGP, talking to an OPNsense router.

<!--more-->

```
~% kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   cilium-8gs69                       1/1     Running   3          4d20h
kube-system   cilium-chnzb                       1/1     Running   3          4d20h
kube-system   cilium-k86sc                       1/1     Running   4          4d20h
kube-system   cilium-operator-65496b9554-64bwh   1/1     Running   3          3d10h
kube-system   coredns-7789cccdc4-czj2z           1/1     Running   2          2d11h
kube-system   coredns-7789cccdc4-vtrfd           1/1     Running   1          35h
kube-system   etcd-node06                        1/1     Running   4          5d20h
kube-system   hubble-relay-7b8fb45847-976xn      1/1     Running   17         5d11h
kube-system   kube-apiserver-node06              1/1     Running   4          5d20h
kube-system   kube-controller-manager-node06     1/1     Running   4          5d20h
kube-system   kube-proxy-2pq5z                   1/1     Running   3          5d20h
kube-system   kube-proxy-6vtpf                   1/1     Running   4          5d20h
kube-system   kube-proxy-wlz6x                   1/1     Running   4          5d20h
kube-system   kube-scheduler-node06              1/1     Running   4          5d20h
```

"Production" use of this cluster for services at home isn't there yet because I
need to first learn:

1. how to upgrade the cluster
2. how to fix a broken cluster
3. what good persistent storage backend is feasible
4. how to preserve source IP addresses as traffic passes through the load
   balancer VIP

I was trying to use Calico because it's what I commonly see with tutorials. And
then Cilium was brought into the picture when I saw it included in EKS Anywhere.
Since $dayJob has EKS Anywhere in the roadmap, I might as well _try_ to mimic
that configuration and eventually deploy an actual EKS Anywhere cluster at home.

I'm probably years behind the curve now but better late than never.
