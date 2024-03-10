---
layout: post
title: Fixing modem jitter for good
image: "/files/header-gpon.jpg"
---

![](/files/header-gpon.jpg)

Frustrated with seeing so much jitter and loss in my homelab, I decided to
replace the modem with a different unit. Didn't want to get the provider
involved, there's no guarantee they'll give a different model and I'm not really
in the mood in explaining this specific problem with support.

<!--more-->

---

Part one: [Fiberhome modem jitter and packet loss](/2024/01/06/fh-onu-jitter-loss.html).

---

After searching around, I ran into great resources at <https://hack-gpon.org>
and [Anime4000's repository](https://github.com/Anime4000/RTL960x). [ODI
DFP-24X-2C2 SFP GPON ONU](https://hack-gpon.org/ont-odi-realtek-dfp-34x-2c2/)
seems to be the model of choice: well documented, used by lots of people, and
compatible across different ISPs. 2.5GbE is available when installed into
capable equipment. This is also the most accessible device in my part of the
world.

There were issues as it was being configured: the ONU was looping between states
`O2`->`O5` and I initially did not have access to the data VLAN. Luckily enough
my unit was not banned with all the tinkering.

# `O2`->`O5` loop

There are two modes for this ONU: Internet Gateway Device (IGD)/Home Gateway
Unit (HGU) or Switch Fabric Unit (SFU)[^onumode].

IGD is commonly known as "router mode" where the device acts as a full fledged
consumer gateway to the ISP network, complete with DNS, DHCP, and other core
network services. Some VLANs are untagged as it leaves the ONU, depending on
provisioning. Transparent bridging is still possible by not configuring any WAN
links within the ONU.

SFU acts as a plain switch (bridge) to the network and passes all traffic
downstream as-is.

The cause is not exactly known, but PLDT's OLT appears to deauthenticate the ONU
when it uses SFU mode. The workaround is to flash IGD/HGU mode firmware. If
you're still not `O5` by then, check your other parameters and reboot the ONU
each time.

My guess is the upstream expects a four port device, where port 1 is for data
and ports 2-4 for IPTV. SFU mode might be presenting itself as a single port
device and the OLT kicks it out because of this mismatch.

More investigation is needed.

# No access to data VLAN

After flashing the GPON stick to HGU firmware, I still did not have internet
access. It was in `O5` but the traffic I saw with `tcpdump` was not typical.
There were lots of ARP packets with unusual addresses at 1.0.0.0/8. Manually
assigning my computer with a VLAN tagged network and activating it caused a
flood of multicast traffic.

This must be the IPTV network. And if I'm reading it correctly it's also on the
same VLAN? Doesn't seem to be right.

After reading through all the available config variables with `flash all`, I
came across `WAN_PHY_PORT` in its default setting of `4`. Setting that to `0`
and rebooting took me out of the IPTV network. I presume this setting assigns
which port in the ONU is assigned downstream.

I expected all VLANs were tagged except it was untagging the internet VLAN
by default. It was only when my desktop received router advertisements and had a
/128 DHCPv6 address assigned I had noticed this was the case and internet access
was available.

I asked for an IPv4 address and was given one. Neat. I wanted to have all the
available VLANs tagged so my router would deal with it, but this'll work for
now. QinQ would be nice too except my switch didn't support it (didn't think
it'd have a use case before buying).

# Finally

It can be concluded the stock modem was causing problems. Recursive DNS traffic
no longer kills latency with the SFP GPON stick in use. I did lose POTS service
in the process but it should be doable with a SIP server and access to the voice
VLAN. Landline is barely used so it's not really a breaking issue. It is useful
when it's needed and available.

[^onumode]: <https://github.com/Anime4000/RTL960x/tree/main/Firmware/DFP-34X-2C2#sfu-mode>
