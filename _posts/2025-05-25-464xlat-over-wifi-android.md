---
layout: post
title: 464XLAT with Samsung Android over WiFi
---

tl;dr: still missing with OneUI 7 or I'm missing something.

I've been trying to transition into an IPv6-mostly network for a year now. Most
of my network infrastructure at home is ready to go IPv6-only except for one
critical blocker: Samsung Android does not have CLAT when on WiFi.

Apple devices work great and the conditions to enable 464XLAT on WiFi for both
desktop and mobile are well known: set a timer on DHCP option
108[^ipv6-only-preferred] and have DNS64 synthesize an IPv6 address for
`ipv4only.arpa`. Samsung Android on the other hand has no documented method that
I know of enabling 464XLAT on WiFi. The same network options were enabled for
Android and while the phone forgoes the IPv4 address, no CLAT was brought up.
This should be fine except it breaks VoWiFi, a critical service where cell
reception is not good.

OPNSense 25.1 ships `radvd` 2.20 which includes `PREF64`[^rfc8781]. While not
presented in the GUI, it is possible to temporarily add the configuration option
`nat64prefix <NAT64 prefix>/<prefixlength> {};` over SSH and sending `SIGHUP` to
`radvd`. Except still no CLAT comes up on the Android even with all three
toggles.

I'm not sure what else is needed to enable CLAT for this Android, if this is
even available at all.

[^ipv6-only-preferred]: <https://www.rfc-editor.org/rfc/rfc8925.html>
[^rfc8781]: <https://www.rfc-editor.org/rfc/rfc8781>
