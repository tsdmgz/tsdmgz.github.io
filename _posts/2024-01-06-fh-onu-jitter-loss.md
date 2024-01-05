---
layout: post
title: "Fiberhome modem jitter and packet loss"
---

For a long time severe jitter and occasional packet loss plagued my internet
connection. It was more puzzling when the problem followed across different
router platforms I've used: EdgeOS, OpenWRT, pfSense, and OPNsense. There are
cases when jitter and loss are bad enough to trigger a failover because it was
failing connection liveliness tests.

I couldn't identify what was causing it for a long time.

<figure>
<img src="/files/grafana-jitter-1.png" alt="Jitter as shown by the spikes in
this latency graph"/>
<figcaption>Jitter as shown by the spikes in this latency graph</figcaption>
</figure>

# Connections?

My first thought was maybe the router didn't like a lot of connections opened at
the same time?

For a quick test, I ran [dachad/tcpgoon](https://github.com/dachad/tcpgoon) and
TCP connection spammed a remote server I spawned on the internet all the way up
to 10,000 connections.

Nothing. Router didn't flinch and has 12GB of memory. Processor utilization was
well below 10%. `pf` State table use was hovering at 1%. Packets per second was
well within typical rates.

# Side quest

I wanted to warm up my local DNS cache for fun and came across
[saint-lascivious/haha_cache_go_brr](https://github.com/saint-lascivious/haha_cache_go_brrr/).
`haha_cache_go_brr` downloads a list of the top 1M domains from [Majestic
Million](https://majestic.com/reports/majestic-million) then throws queries to a
target nameserver at a configured rate.

Since this was a one-off run, manually running the script's commands was good
enough. Only the first 3500 domains were queried at ~10 parallel lookups at a
time[^dns-command]. We don't want to set off pages for any admins.

# It was DNS?

Shortly after running the cache warmup script, I noticed the latency graph
wobble. That's a potential investigation path.

To verify it was triggered with recursive lookups, I ran a DNS stress test
(forgot which) against my local resolver at around more than 3000 queries per
second with locally defined names. These lookups don't go out to the
internet.

No effect.

Here's what we have:
* It's not the connection count
* It's not packet rates
* It's not local name resolutions
* It's not triggered entirely within the network

Evidence strongly suggest recursive lookups are the trigger.

<figure>
<img src="/files/grafana-jitter-2.png" alt="Graph of recursive DNS requests per
second. These spiky bars are from organic lookups from devices within the
network."/>
<figcaption>Graph of recursive DNS requests per second. These spiky bars are from
organic lookups from devices within the network.</figcaption>
</figure>

<figure>
<img src="/files/grafana-jitter-1.png" alt="Jitter as shown by wobbles in
this latency graph"/>
<figcaption>Jitter as shown by wobbles in this latency graph</figcaption>
</figure>

These graphs conveniently line up. Let's verify.

Another manual run made the DNS graphs respond appropriately:

<figure>
<img src="/files/grafana-jitter-4.png" alt="Rising amount recursive DNS requests
as uncached requests are received"/>
<figcaption>Rising amount of recursive DNS requests as uncached requests are
received.</figcaption>
</figure>

and then:

<figure>
<img src="/files/grafana-jitter-3.png" alt="Latency graph as the local DNS cache
was being warmed up"/>
<figcaption>Latency graph as the local DNS cache was being warmed
up.</figcaption>
</figure>

I ran the test at 15:15 against one local recursor. The network experienced
packet loss and jitter even when pinging from my desktop. Another test was run
from the desktop to the bundled Unbound service on the router. While these
queries are not measured because this was not instrumented, its effects were
still seen on the latency graph. On the third test, lookups were run directly on
the router querying the locally running Unbound service. Results were consistent
with previous tries. Subsequent tests followed the same pattern but with
connection failover disabled.

On another test (not shown), Unbound was configured to use TCP-only lookups with
`tcp-upstream: yes`. No problems were shown, or it was somehow rate limited
because of all the connection handshakes it had to do. In either case, no
latency fluctuation was seen[^dns-tcp-perf].

<div style="margin: auto; width: 30%;">

<blockquote>0 days since it was DNS</blockquote>

</div>

Maybe not entirely DNS. There are unexplored conditions:
1. Plenty of TCP connections to *different* servers
2. Plenty of UDP connections to a single server with different src:dst ports
2. Plenty of UDP connections to *different* servers

My guess is that recursive lookups generate a lot of UDP traffic to several
different destinations in short bursts, fulfilling the third condition. We'll
focus on that.

# Walking up the path

Router and internal network are mostly ruled out. We are left with the modem.
Which is bizarre because this was configured to bridge mode. It's not supposed
to be affected by traffic aside from raw packet throughput.

Getting information from the modem is difficult. Information is sparse, the ISP
configured firmware stripped out almost all configuration options. Normally,
bridge mode negates that condition because layers two and three is passed
through to the router downstream. Configuring the modem is mostly not needed in
this case.

It appears to be so bad, a single recursive lookup can wobble the whole graph.

# Fix?

Either I convince PLDT to send me a new modem with a different vendor or I order
one of those GPON-on-a-stick modems.

Good luck to us in getting this escalated to tier 2 technical support.

I'll probably come back once I obtain that GPON-on-a-stick and replace that
FiberHome ONU.

[^dns-command]: `cut -f3 -d, majestic_million.csv|head -n3500|xargs -P 10 -n1 dig @<unbound> +short +time=7 +tries=2`
[^dns-tcp-perf]: Significant performance loss is seen with TCP-only upstream.
    Avoid unless network conditions require this setting.
