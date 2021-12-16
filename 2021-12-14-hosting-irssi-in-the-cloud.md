# Hosting Irssi in the Cloud

This post goes into the journey and reasons of why and how I am currently
hosting the IRC client Irssi in my Kubernetes Cluster.

## Permanently Connected

If you use IRC you know it is a badge of honor or rite of passage to have your
nick always be online, and ideally also have access to the log of messages that
happened while you were gone. There are multiple ways to achieve this, many of
which I've tried.

### ZNC Bouncer

I've been running a ZNC bouncer on my Raspberry Pi, and connecting to the IRC
client to that, running the IRC client itself in GNU screen on the Pi, and most
recently, using [Quassel IRC](https://quassel-irc.org) in my Kubernetes cluster.

Anything that runs on my Pi suffers from the fact that my internet tends to go
down at least once a day (it seems to be the router itself, almost exactly every
24 hours), meaning that my nick would reconnect in the IRC channels in which it
is also at least once a day, to the annoyance of some others in the channels,
and to my displeasure regarding a perfect uptime.

### Quassel IRC

This lead to me installing Quassel Core onto my Kubernetes cluster. This is
similar to a bouncer, as in the core is permanently connected to the IRC
networks, however, you also need a Quassel Client, which communicates to the
core via its own protocol. This should already point out the first disadvantage
of Quassel, in my opinion. I'd much rather have my IRC chat in an aesthetic
terminal using the IRC client of my choice than running a potentially ugly GUI
application. The official Quassel Client can at least be styled with CSS, but
it's still not what I would prefer.

The final straw was however a yet unexplained phenomenon that started happen one
day out of the blue.

#### Memory Leak

As can be seen in the following screenshot of cluster resources over seven days,
the Quassel deployment seemed to have a memory leak.

![Grafana](https://user-images.githubusercontent.com/9076894/146248943-944d853a-1503-4f5d-94ca-ceb0f160eb3f.jpg)

The past four days show an accumulation of memory in the Quassel pod, right up
until the entire node runs out of memory. This could be fixed by pod eviction or
setting a memory limit, but that would only be a band-aid rather than a root
cause analysis. Simultaneously, the CPU resources of cilium, the network
driver, ramp up, which has to be related to the memory increase.

I realized that I had a three year old image tag of Quassel deployed, but
upgrading that did not help, alas.

## Moving On / Combining the Best of Both Worlds

In any case, this gave me a good excuse to move away from Quassel, in addition
to the aforementioned reasons. Since Irssi has been my favorite client so far,
the only disadvantages stemming from the Pi setup, I set out to deploy Irssi in
Quassel's stead in my cluster.
