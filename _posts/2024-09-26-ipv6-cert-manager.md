---
layout: post
title: Some notes in setting up cert-manager with Cloudflare and DNS-01
---

I've been bringing up a full fledged Kubernetes cluster using Raspberry Pis with
features expected from a typical cloud provider for the past months.

It is not easy.

I've run into some complications with [cert-manager](https://cert-manager.io)
and ACME when pod and service networks are configured for single stack IPv6.
Specifically in this configuration:
* IPv6-only cluster
* Cloudflare as authoritative DNS
* Let's Encrypt wildcard certificate
* DNS-01

<!--more-->

Before starting on anythihg, know the certificate's lifecycle and step through
the process of a request. Cert-manager's troubleshooting documentation is a
great starting point[^3].

---

# Gotcha no. 1: Asking for a certificate with the same FQDN as the cluster

```
E0923 15:28:27.941775       1 sync.go:190] "propagation check failed" err="DNS record for \"app.site.example.com\" not yet propagated" logger="cert-manager.controller" resource_name="gateway-1-762547344-396123462" resource_namespace="default" resource_kind="Challenge" resource_version="v1" dnsName="app.site.example.com" type="DNS-01"
```

By default, cert-manager uses the pod's resolver. Typically, this is
`resolv.conf` pointing to the cluster's DNS server (typically CoreDNS) and it
works almost all the time. Until the requested certificate is
`*.app.site.example.com` and your cluster is configured to use the domain
`app.site.example.com`.

Part of cert-manager's workflow in a DNS-01 request is to check if the ACME
challenge has propagated to DNS by looking up for `TXT` record
`_acme-challenge.app.site.example.com`. The default configuration points to the
cluster's DNS resolver, and the cluster is configured to use
`app.site.example.com`, therefore CoreDNS will answer because the query is
within its configured domain.

To solve this, configure cert-manager with a different recursive nameserver.

Helm:

```
dns01RecursiveNameservers: "192.0.2.1:53,198.51.100.1:53"
dns01RecursiveNameserversOnly: "true"
```

Or edit the deployment and add these arguments:

```
spec:
  template:
    spec:
      containers:
      - args:
        - --dns01-recursive-nameservers-only
        - --dns01-recursive-nameservers=192.0.2.1:53,198.51.100.1:53
```

Credit to [@peske](https://github.com/peske) in
[cert-manager/cert-manager#5515](https://github.com/cert-manager/cert-manager/issues/5515#issuecomment-1479054700)
for reminding about split-horizon DNS explaining this behavior[^1].

Alternatively, CoreDNS may be configured to forward requests specifically for
`_acme-challenge.app.site.example.com` upstream.

Leading us to...

# Gotcha no. 2: IPv6-only, remember?

Telling cert-manager to use 192.0.2.1 and 198.51.100.1[^4] as its recursive
nameservers is a solution. Except the cluster does not provide IPv4 addresses
and 464XLAT is not available, but there is a NAT64 gateway.

The fastest way out is by providing an IPv6 enabled resolver *outside* the
cluster and if it does not have anything specially configured for
`app.site.example.com` zone.

If no IPv6 resolvers are available but NAT64 gateways are present, use the
previous resolvers with IPv4-embedded IPv6 address notation[^2]. This can be
derived by combining the NAT64 prefix and the usual IPv4 dotted decimal format.
For example: using the well-known NAT64 prefix 64:ff9b::/96 and Google's public
IPv4 DNS server, the NAT64 derived address would be `64:ff9b::8.8.8.8`. No
further manual address format conversion is needed as the OS automatically
converts it into the expected IPv6 address for you.

That's all I have so far.

[^1]: https://github.com/cert-manager/cert-manager/issues/5515#issuecomment-1479054700
[^2]: https://www.rfc-editor.org/rfc/rfc6052#section-2.4
[^3]: https://cert-manager.io/docs/troubleshooting/
[^4]: These are example IP addresses, do not use. See [RFC 5737](https://www.rfc-editor.org/rfc/rfc5737.html).
