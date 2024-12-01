---
layout: post
title: Half the traffic, half the pods
---

install crd
install cilium

Three node cluster, using Cilium Gateway API. Four instances on two worker
nodes, 503 half the time.

finding out: cluster trying to get to pod ip over local network with ndp but no
answer

I think I finally found the bottom of my problem:

# Given

Cilium with `ipv6NativeRoutingCIDR: "fdb9:3958:b1b1:040b::1:0/96"`, and
`kubernetes` service at `fdb9:3958:b1b1:040b::2:1`. A `LoadBalancer` service
from GatewayAPI was created using a _different_ `ClusterIP` whose `ExternalIP`
is _also_ at `fdb9:3958:b1b1:040b::2:1`.

CoreDNS is unable to connect to `kubernetes` service and errors out with
```
[ERROR] plugin/kubernetes: pkg/mod/k8s.io/client-go@v0.29.3/tools/cache/reflector.go:229: Failed to watch *v1.Namespace: failed to list *v1.Namespace: Get "https://[fdb9:3958:b1b1:40b::2:1]:443/api/v1/namespaces?limit=500&resourceVersion=0": tls: failed to verify certificate: x509: cannot validate certificate for fdb9:3958:b1b1:40b::2:1 because it doesn't contain any IP SANs
```

# When

Some time has passed, pods lose communication with each other. `cilium-dbg
status` reports reachable nodes but unreachable endpoint failures. On a two node
worker cluster, half of the requests to the gateway fail because the request
goes to a now-stale IP address.
