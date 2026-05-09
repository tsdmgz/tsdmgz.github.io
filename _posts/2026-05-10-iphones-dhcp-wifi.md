---
layout: post
title: iPhones, WiFi, DHCP
---

I noticed iOS devices dropping out of WiFi and switching to mobile data when
unused for around an hour or so, reconnecting to WiFi when woken up. This is a
source of frustration as cellular coverage indoors isn't the best and drains our
devices' batteries faster. VoIP calls like with Signal or Facebook often fail to
connect or don't connect quickly enough due to poor connectivity. Android
devices are fine and don't exhibit the same behavior.

<!--more-->

WiFi Assist is disabled. Disabled cellular data to see if it was very silly
behavior from iOS. Checked everything else related to WiFi or cellular data. No
difference in anything.

<figure>
<img src="/files/iphones-dhcp-wifi-01.png" alt="Graph snapshot of the iPhone in
question irregularly dropping in and out of the network in a two hour span."/>
<figcaption>Graph snapshot of the iPhone in question irregularly dropping in and
out of the network in a two hour span.</figcaption>
</figure>

The way disconnections and connections look like in the graph, it reminded me of
how the IPv4 DHCP server is set to 900 second lease times. A 900 second lease
time means the phone should renew 450 seconds in the lease, and a second time
shortly before the lease expires. Extending the lease time to 3600 seconds
stopped the phone from dropping out of the network completely.

My guess is the iPhone goes to sleep for an extended period and does not send a
renewal on time. Extending the lease period may have let the phone overlap
between its periodic wake cycle and renew its lease on time.
