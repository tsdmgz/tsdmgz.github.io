---
title: CUPS in a container with AirPrint and AirScan
layout: post
---

My network has a Raspberry Pi running CUPS and
[AirSane](https://github.com/SimulPiscator/AirSane) as a network print and scan
server for a USB-only Epson L360. Until the PoE hat somehow broke and fried the
Pi with it. A spare mini PC was installed as a replacement and thought of a
challenge: can I run AirPrint and AirScan services using only containers?

<!--more-->

# Motivation

I wanted to stay consistent with openSUSE MicroOS's design goal of keeping an
immutable base OS with as minimal changes as necessary. Workloads are stuffed
into individual containers.

Containers also provide consistent environments which can be easily spun up
with a few commands. Configuration is kept as code, trackable with source
control. I don't have to worry about replacing the machine and spend lots of
time reconfiguring.

# What makes it a challenge?

* mDNS
    1. The default Podman or Docker network NATs traffic originating from
       containers and use port forwarding for incoming connections. AirPrint and
       AirScan rely on mDNS for discovery and name resolution. mDNS typically does
       not cross subnets by default because it is multicast and TTL is usually 1.
       Because the container host acts as a router for the containers, only
       colocated containers see mDNS traffic.
* Avahi
    1. Avahi will advertise the container's IP address with the default
       container network configuration. This address is usually inaccessible from
       outside the host, always changes on any restart, and will cause additional
       configuration pain if made static or routable within the network.
    2. Avahi needs direct access to the host network segment for correct
       dispatch of mDNS traffic.
* D-Bus
    1. D-Bus usually uses UNIX sockets and rely on UIDs for authentication.
    2. UIDs are different across containers and will not match expected values.
    3. Container-to-container communication feels like a niche use case.
* CUPS
    1. Direct access to the USB device is needed.
    2. Depends on D-Bus to tell Avahi about its available services.
    3. There were dependencies not included during the installation and caused
       AirPrint discovery to fail.
* AirSane
    1. Same as CUPS.

# Design goals

1. Minimize host-specific configuration as much as possible.
2. Maintain one-process-per-container pattern.
3. Keep container configuration as generic as possible.

It is possible to bundle all of these into a single container, and all processes
inheriting Avahi's requirement of running on the host network stack. This is
fine for CUPS and AirSane but not okay for D-Bus. Reasons discussed later.

In order of dependencies, these need to be started in order:
1. D-Bus
2. Avahi
3. AirSane and/or CUPS

# Solve for X

I'll explain how the services were implemented and what compromises were made
and reasons why.

## D-Bus

D-Bus listens on a UNIX domain socket with authentication by default. Since the
socket exists in the filesystem, the socket can be shared by placing it into a
named volume and mounting the same volume into other containers.

Working around authentication is a little bit more difficult. Because processes
that need to use it are in other containers, UIDs expected by D-Bus may not
match. Auth is disabled by adding the following lines to `session-local.conf`

```xml
<busconfig>
<auth>ANONYMOUS</auth>
</busconfig>
```

Using TCP sockets[^dbus-sockets] and disabling authentication enables cross container
communication. As long as D-Bus's port is not exposed outside the container
network, this should be safe enough at home.

This, in combination with telling other containers to use a different D-Bus endpoint
with `DBUS_SYSTEM_BUS_ADDRESS` should let other applications discover it.

## Avahi

Avahi handles Bonjour for autodiscovery. This requires the host IP address to be
known within the container. The easiest way to provide this is to assign host
networking for this specific container. This is the only directly exposed
service within the stack.

Assigning host networking to the service is provided by `--network=host` within
Podman.

That should handle Avahi.

## CUPS

Letting CUPS know about the D-Bus address was simple: provide
`DBUS_SYSTEM_BUS_ADDRESS` pointing to the instance.

Working with SELinux, detecting the correct device, and assigning the correct
permissions made it a little more difficult. In fact, I haven't even solved
assigning correct permissions yet. Working around the first two invoved giving
`--privileged` permissions for CUPS's container.

Because UIDs and GIDs are remapped between container and host, permissions for
the printer's device node were changed to world read-write: `chmod 0666`. And
because this was a privileged container, it can also reassign permissions on
other device nodes.

I was not happy with that. There are ways to correctly map UIDs and GIDs but
it's good future exercise in narrowing down to least privileges needed.

Then another issue.

For some reason, the printer wasn't being detected even when mDNS broadcasts
were sent out, as verified with Wireshark. What was more perplexing was it only
occurs when using Alpine Linux as the base OS. Other distributions like openSUSE
Tumbleweed worked as expected.

I discovered it required the packages `nss` and `nspr` in Alpine for the printer
to be detected. I don't exactly know why yet. Once those were installed, the
printer was happily detected by all devices in the network.

# Proof of concept

I've probably skipped a few things by accident but it should be enough to get
the stack going.

[tsdmgz/container-cups-airsane](https://github.com/tsdmgz/container-cups-airsane)
contains everything discussed here as code.

# Points for improvement

* Run CUPS and AirSane without `--privileged`
* Have D-Bus listen on a UNIX domain socket instead of TCP (is this possible?)
* Move to rootless containers

# Footnotes

[^dbus-sockets]: <https://dbus.freedesktop.org/doc/dbus-specification.html#transports-tcp-sockets>
