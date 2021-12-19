# Hosting Irssi in the Cloud

This post goes into the journey and reasons of why and how I am currently
hosting the IRC client [Irssi](https://irssi.org/) in my Kubernetes Cluster.

## Permanently Connected

If you use IRC you know it is a badge of honor or rite of passage to have your
nick always be online, and ideally also have access to the log of messages that
happened while you were gone, as this is not the case by default in IRC,
contrary to many new messaging platforms such as Discord. As I've pointed out
many times (ironically?), IRC is far superior to Discord in that regard.

There are multiple ways to achieve "always online", many of which I've tried.
The first part of this post goes into the history of past setups. The current
solution is described at the end.

### ZNC Bouncer

I've been running a ZNC bouncer on my Raspberry Pi, and connecting to the IRC
client to that, running the IRC client itself in GNU screen on the Pi and SSHing
to it, and most recently, using [Quassel IRC](https://quassel-irc.org) in my
Kubernetes cluster.

Anything that runs on my Pi suffers from the fact that my internet tends to go
down at least once a day (it seems to be the router itself, almost exactly every
24 hours), meaning that my nick would reconnect in the IRC channels in which it
is also at least once a day, to the annoyance of some others in the channels,
and to my displeasure regarding a perfect uptime.

### Quassel IRC

This led to me installing Quassel Core onto my Kubernetes cluster. This is
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
until the entire node runs out of memory (~1.5G/2G). This could be fixed by pod
eviction or setting a memory limit, but that would only be a band-aid rather
than a root cause analysis. Simultaneously, the CPU resources of cilium, the
network driver, ramp up, which has to be related to the memory increase.

I realized that I had a three year old image tag of Quassel deployed, but
upgrading that did not help, alas.

## Moving On / Combining the Best of Both Worlds

In any case, this gave me a good excuse to move away from Quassel, in addition
to the aforementioned reasons. Since Irssi has been my favorite client so far,
the only disadvantages stemming from the Pi setup, I set out to deploy Irssi in
Quassel's stead to my cluster.

In essence, I wanted to emulate the second setup I had going on on the Pi, just
inside a container. Fortunately there is an official [Irssi container image](https://hub.docker.com/_/irssi).
I initially chucked that into a
[Statefulset](https://github.com/LoLei/dotfiles/blob/master/.irssi/k8s/statefulset.yaml)
Kubernetes resource (statefulset rather than deployment since arbitrary replicas
don't make much sense), attached my configuration files in a persistent volume,
and voilà, it's running again.

### Security Context

One thing to point out, I initially could not edit the and copy the config files
to the volume due to permission errors. This was fixed by adding a
`SecurityContext` to the pod spec:

```yaml
securityContext:
  fsGroup: 1000
```

The `fsGroup` corresponds to the `user` user that is created in the container
image.

### Containerfile

I based a custom [Containerfile](https://github.com/LoLei/dotfiles/blob/master/.irssi/k8s/Containerfile)
(aka Dockerfile) off of the official Irssi one, since I needed to install GNU
screen. This image also starts with an endless sleep loop by default, while Irssi
can be started manually by attaching to it. I haven't made my mind up if Irssi
should be started automatically. In theory it might be beneficial since then on
a crash, it could automatically reconnect, however I haven't set up the Irssi
config to do that yet, also, the likelihood of this pod exiting is very slim. I
might have said the same about the Quassel deployment, and would've been proven
wrong, so I may still opt for that route in the future.

### Future Endeavors

On my Pi SSH setup, I used [notossh](https://github.com/guyzmo/notossh) to
forward Irssi notifications from the Pi to the system that is SSH'd to the Pi. I
am yet unaware if such a solution can also be used for this new setup, however
in the meantime I am content with using the
[hilightwin plugin](https://github.com/irssi/scripts.irssi.org/blob/master/scripts/hilightwin.pl),
which collects mentions in a specific Irssi window. In the future though, I may
look into forwarding notifications as well.

### Why Kubernetes

One might argue that running a simple TUI application in Kubernetes is overkill,
and I'd be inclined to agree. The only benefit I can immediately see is the
automatic restart and reconnect, if I were to enable that in Irssi. However, the
container component of that can just as well easily be accomplished with a
docker compose setup. So the reason I chose the setup I did is only because I
have the cluster anyway, and this deployment barely takes up any resources.

## Conclusion

I am still not 100% satisfied with my setup, since now it seem overly convoluted
for such a simple use case, but in my opinion still the best out of the previous
arrangements, if the Pi connection wasn't so unstable. If I decide to yet
again change it up, a blog post will certainly follow at some point, so be
sure to stay subscribed. Or just check the site periodically, since there is
no RSS feed (yet).
