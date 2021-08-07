# My Own Kubernetes Dashboard

I recently made my own dashboard displaying some information about my Kubernetes cluster. In this
post I will gather some thoughts about the reasoning of why I built it, and some information about
the technical details.

## Why - History

I wanted to have a dashboard for my Kubernetes cluster, so I can see and monitor various information
about the infrastructure at a glance, without having to have access to a terminal or the cloud
provider interface.

The most popular choice for this is [Grafana](https://grafana.com/), along with the
[Prometheus Stack](https://github.com/prometheus-operator/kube-prometheus). Prometheus and the
[Metrics Server](https://github.com/kubernetes-sigs/metrics-server) gather information about the
infrastrcture and act as data sources with which Grafana can interface and use as a resource.
Grafana displays this data in a nice way.
[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) can be used to trigger
alerts (notifications, emails, Slack channel posts, …) based on certain criteria.

All of this is amazing, an industry standard, even essential for some commercial infrastructure
deployments. However, it comes with a huge downside for small hobby deployments, as my cluster is.
I set up all of the above described components, and if you're lucky it's still up at
[grafana.lolei.dev](https://grafana.lolei.dev). After the setup I immediately noticed the vast
resource requirements of it. The cluster's memory usage jumped up from about 25% to up to 90%. The
CPU usage increased as well but not as drastically. kube-prometheus-stack handles a lot of data,
which explains the increased usage. The irony is however that it uses way more resources than all
the other deployments it monitors, even combined at some points. This may not be as noticeable or
even an issue on clusters with more resources, but since at this stage the nodes are very minimally
outfitted, it presents a considerable downside, allowing less room for other deployments.

![Screenshot 2021-08-06 at 18-03-31 Kubernetes Compute Resources Cluster - Grafana](https://user-images.githubusercontent.com/9076894/128595750-6f11b220-d7dd-4d92-b9bc-f821f5e95f0a.png)

The above image shows a recent screenshot of my Grafana dashboard. The blue part is the
kube-prometheus-stack namespace, including all of the components mentioned above.

These large resource requirements led me to the idea to implement my own, more minimal dashboard
interface. The initial source leading me to the right technologies came from [this
article](https://learnk8s.io/real-time-dashboard). I implemented a similar dashboard, but from
scratch, with a different user interface and swappable backend, the ins and outs of which are
described in the next section.

## How - Technology

The dashboard is split up into two main components/deployments: Frontend and backend. The frontend
is the only publically exposed service, and calls the cluster-internal backend service via the
Next.js middleware API routes, which run in the Next.js webserver, as opposed to the browser client.
This makes it easy to swap out the backend service by just specifying a different cluster-internal
service hostname in the frontend deployment configmap.

### Backend

Instead of sending raw requests to the Kubernetes API, a client wrapper is used.

The backend has been implemented with (so far):

- Javascript (Typescript) client: [kubernetes-client-js](https://github.com/kubernetes-client/javascript)
  - Most everything works out of the box, it was enjoyable experience using it. This may be due to
    the fact that is is a client "officially" made by the Kubernetes team.
- Rust client: [kube-rs](https://github.com/clux/kube-rs) (Currently deployed)
  - This client had some issues, which made it overall a less enjoyable experience, but is **is**
    noticeably faster than the Typescript client.
  - Particularly the [topNodes](https://github.com/kubernetes-client/javascript/blob/6b713dc83f494e03845fca194b84e6bfbd86f31c/src/top.ts#L20)
    function from the JS client had to be re-implemented, to get node-wide resources. (This would
    not have been necessary if the metrics API was supported.)
  - Resources are represented in [Quantities](https://docs.rs/k8s-openapi/0.12.0/k8s_openapi/apimachinery/pkg/api/resource/struct.Quantity.html),
    which are byte strings. The `AsInt64()` function was apparently not ported from Go, therefore
    doing arithmetic with human-readable bytes strings got a tad ugly as well.
  - The default non-cluster deployment via the kube config file does not work and needs a [separate user](https://github.com/kube-rs/kube-rs/issues/196).
  - [Another workaround](https://github.com/kube-rs/kube-rs/issues/587#issuecomment-877314745) is needed to be able to run it within the cluster.

Why did I even implement the backend twice? The main reason is that I wanted to try swapping out the
backend on-demand via configuration variables in the frontend. And since the Typescript client
worked so well, I wanted to implement a more ambitious version as well.

Regardless of which backend version is used, to view pods and nodes within a pod in the cluster, a
different serviceaccount than the default one had to be created and assigned to the pod, along with
a clusterrole and clusterrolebinding granting the correct RBAC permissions (see the k8s directory).
**Cluster**role because nodes are not namespaced, and pods in other namespaces should be viewable as
well.

### Frontend

Next.js with its [API routes](https://nextjs.org/docs/api-routes/introduction) is used in order to
be able to access the cluster-internal backend service. The UI does not look as pretty as Grafana,
and is naturally far less fully featured, however it works reasonably well, both on desktop and
mobile.

## Conclusion

Was it worth it? It did not take that long, exposed me to some new Kubernetes information I would
not have learned otherwise (or not as soon), was fun (at some parts), and gave me a dashboard that
does not use insane resources. So overall I would say yes, it was worth it.
