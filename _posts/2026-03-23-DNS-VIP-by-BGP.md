---
layout: post
title: DNS VIP by BGP
---

Kubernetes backed DNS resolver. VIP is provided by BGP. This is great. Then you
start having loose cables and now your BGP sessions aren't recovering. The whole
house is annoyed. Now another port is flapping. _You_ are getting more annoyed.

Friends, don't run your DNS VIP by BGP unless you have a dedicated room for the
network. It's not fun.
