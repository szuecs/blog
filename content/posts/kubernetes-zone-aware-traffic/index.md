+++
title = "Index"
date = 2026-04-18T20:02:25+02:00
images = []
tags = ["kubernetes", "infrastructure", "distributed systems"]
categories = []
draft = true
+++

Today, I want to share a bit of details in our steps to support zone
aware traffic in our [Kubernetes infrastructure](https://kubernetes-on-aws.readthedocs.io/en/latest/admin-guide/kubernetes-in-production.html).

You can learn a bit of [Kubernetes](https://kubernetes.io/) and
interesting effects it can have. First I have to share something about
the environment.

There are basically two patterns in Kubernetes infrastructure deployments:

1. 1 Kubernetes cluster per zone
2. 1 Kubernetes cluster per region with multiple zones.

In our case it's the latter, a single Kubernetes cluster spans
multiple zones, 3 by default. The advantage is that you have more
simple availability possbilities, if applications run across 3 zones
by default. On the other hand, doing traffic engineering, so prefer
close instances is more complex. Before I tell about the problem space
and our findings, let me share our basic [Kubernetes
Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
setup. It is a 2-layer load balancer infrastructure serving a large
scale microservice environment.

In German there is a proverb that says one picture is better than
thousands of words, so let's see figure 1. It's a bit outdated,
because we run AWS network load balancer (NLB) instead of ALB, but
rest looks the same today.

{{< figure
	src=ingress-traffic-flow-aws-technical.svg
	alt="2-layer load balancer traffic flow from AWS ALB to skipper to application pods"
	link="https://opensource.zalando.com/skipper/kubernetes/ingress-controller/#aws-deployment"
	caption="figure 1"
>}}

We terminate TLS in the cloud load balancer and the HTTP routing is
done in our HTTP proxy [skipper](https://github.com/zalando/skipper).
Cloud load balancers are managed by [kube-ingress-aws-controller](https://github.com/zalando-incubator/kube-ingress-aws-controller)
and DNS by [external-dns](https://github.com/kubernetes-sigs/external-dns).

We run this for about a decade since Kubernetes 1.3 and it evolved
quite a bit. If you want to lurk into our configuration, check it out
in [kubernetes-on-aws](https://github.com/zalando-incubator/kubernetes-on-aws/tree/dev/cluster/manifests/skipper).

In this environment zone aware traffic has 4 parts:

1. client to cloud load balancer
2. cloud load balancer to skipper-ingress (skipper-ingress is the application based on skipper that is the data-plane and control-plane of our http proxy layer)
3. skipper-ingress to backend pods
4. cluster internal clients to skipper-ingress through a service type ClusterIP

1. and 2. are done by kube-ingress-aws-controller. You can set
`-nlb-cross-zone=false` to disable sending traffic cross zone from the
load balancer TargetGroup to skipper-ingress.  If you set
`--nlb-zone-affinity=availability_zone_affinity` all clients running
in the same zone in any AWS account will resolve DNS to same zone. So
instead of 3 IPs for NLB TLS Listeners you will resolve only one. It
does not matter if you have a client in the same or in another
cluster.

Since
[v0.24.22](https://github.com/zalando/skipper/releases/tag/v0.24.22)
skipper supports zone aware traffic in its kubernetes
dataclient. Dataclients are the way to fetch data, that skipper uses
to create its routing table. If you run skipper-ingress with the
kubernetes dataclient configured, every skipper-ingress pod will fetch
all relevant Kubernetes objects to build its routing tree. In larger
environments this is quite some load on the Kubernetes control plane
and it's easy to break Kubernetes control plane by scaling out. The
way to control the load that is done to the Kubernetes control plane
is to run skipper's `routesrv`.
[Routesrv](https://opensource.zalando.com/skipper/kubernetes/ingress-controller/#routesrv)
is a skipper component and uses the Kubernetes dataclient to fetch
routing information and exposes an API endpoint to fetch eskip
routes. [Eskip](https://pkg.go.dev/github.com/zalando/skipper/eskip)
is the skipper native routing language. Of course if your data-plane
skipper-ingress fetches routes from a control plane component like
routesrv, the question is: how does routesrv know where the data-plan
is running?

The answer in our routesrv based zone aware traffic feature available
in [v0.24.64](https://github.com/zalando/skipper/releases/tag/v0.24.64)
is: it does not!

It exposes more route API endpoints:

- `/routes` fetch all routes with all ready endpoints
- `/routes/:zone` fetch zone aware routes

Skipper-ingress data plane pods get its zone by Kubernetes downwards
API, which makes it possible to pass Kubernetes metadata to the
process by environment variables:

```yaml {linenos=inline style=emacs}
env:
- name: KUBE_NODE_ZONE
  valueFrom:
    fieldRef:
      fieldPath: metadata.labels['topology.kubernetes.io/zone']
```

Now you can use the environment variable in a flag to skipper like
`-routes-urls=http://skipper-ingress-routesrv.kube-system.svc.cluster.local/routes/$(KUBE_NODE_ZONE)"` to fetch zone aware routes.

Great now we understand how we can do zone aware traffic 1.-3., but what about 4.?

Ok, 4. seems to be easy. You just plug an annotation
`service.kubernetes.io/topology-mode: auto` and kube-proxy will make
sure your ClusterIP service is having a safe amount of Kubernetes pods
in its layer 4 load balancer. We run this since more than 3 years
without an issue.  Before this annotation there was
`service.kubernetes.io/topology-aware-hints` annotation which did
basically the same thing. If you read the
[documentation](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/),
you see that there are some thoughts about safety, because you do not
want to create harm on a data-plane feature that every application has
to rely on.

As often in a life, things change. Sometimes implementation changes or
your monitoring adds more visibility or application requirements on
the infrastructure change or traffic patterns, because someone
deployed a new client to some service.

To understand better the zonal traffic in our clusters we created a
graph how much RPS by zone we have in skipper-ingress data-plane. If
we have some new data and it shows something unexpected, my first
question is always: can we trust the data?  In this case it seems we
really were able to.  During the last weeks we had some interesting
effects in one of our high traffic clusters, that is shown in
figure 2. We can see that within 1h there were 3 times a large share
of throughput hit only one zone and after some minutes it was going
back to normal.

{{< figure
	src=graph_split_traffic.png
	alt="RPS traffic by zone and we can see a huge traffic split between these within 1-2 minutes and after a while it collapses again and it happened 3 times within 1 hour."
	link="https://opensource.zalando.com/skipper"
	caption="figure 2 - RPS traffic by zone"
>}}

What we see is flapping of the traffic distribution, which sometimes
caused an unexpected latency spike to one of our applications. This
flapping was caused by kube-proxy, that thought it might makes sense
to flap between zone aware and zone unaware traffic. If you check [safeguards](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/#safeguards),
you can read:

    4. One or more endpoints does not have a zone hint: When this happens, the kube-proxy assumes that a transition from or to Topology Aware Hints is underway. Filtering endpoints for a Service in this state would be dangerous so the kube-proxy falls back to using all endpoints.

    5. A zone is not represented in hints: If the kube-proxy is unable to find at least one endpoint with a hint targeting the zone it is running in, it falls back to using endpoints from all zones. This is most likely to happen as you add a new zone into your existing cluster.

Basically what we see in figure 2 is that if zone hints are populated,
kube-proxy will write layer 4 rules such that rules are zone aware and
we have this large unbalanced traffic split. Later zone hints
disappear or are not available for one or more endpoints (4.) and it
decides that it is too dangerous and the rules will change to non-zone
aware traffic. I don't know why these hints disappear, but I did not
find any when I checked. One of the interesting facts are that we have
this flapping all the day for some weeks and it's most often not an
issue, but sometimes a latency spike up to 250ms happened and this is
large enough for high traffic low latency applications to fail.

After discussing this in Kubernetes sig-network community channel we
will try to switch to a more persistent `trafficDistribution: PreferSameZone`,
that is now available in Kubernetes. It will provide no flapping for
the traffic distribution. I am looking forward to see the effects.

One other important configuration is that you have a balanced spread
of pods for clients, proxy and backends. This you can influence by setting
[`topologySpreadConstraints`](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/).
The application developers already enabled `topologySpreadConstraints`,
example:

```yaml
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              component: ingress
              application: skipper-ingress
```

This is important to not have such unbalanced traffic created by
clients and also that targets of the proxy are also spread evenly so
the horizontal pod autoscaling can do its job and keep the load of
single pods in bounds.

Of course Kubernetes would not be Kubernetes, that everyone loves and
hates, if there would not be a missing feature. Beware about the fact
that there is no [zone aware down scaling](https://github.com/kubernetes/kubernetes/issues/124149).

Kubernetes infrastructure is super interesting, often details matter
and sometimes there are missing features, that make you wonder, but
all in all I am very happy with it.

If you have any questions or anything to share let me know in [Mastodon](https://hachyderm.io/@sszuecs).
