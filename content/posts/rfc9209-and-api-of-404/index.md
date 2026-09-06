+++
title = "RFC9209 and the API of a 404"
date = 2026-07-09T22:30:38+02:00
images = []
tags = ["proxy", "rfc", "api", "infrastructure", "kubernetes", "incident"]
categories = []
draft = true
+++

We have an interesting [public debate](https://github.com/zalando/restful-api-guidelines/pull/869)
on the status code `404` in Zalando's API guidelines. I am a clear
opponent to a change the proxy response from `404` to `5xx`, but let's
try first to understand why they want to change.

The reason why this change proposal started was an incident, that our
[canary](https://github.com/zalando-incubator/kubernetes-on-aws/blob/dev/cluster/manifests/skipper/deployment.yaml#L4)
had some miss configuration. The canary is one pod in our ingress
fleet. This should mitigate the blast radius of updating our ingress
data- and control-plane.  However the data-plane started successful,
but had no routes created by Kubernetes ingress and routegroup
objects, so it responded `404` to all requests directed to applications.
So as expected we had something like `<5%` error rate responding `404`
instead of reaching the applications, until some other control loop
detected that we have no active applications anymore. The mentioned
control loop got the connection to the skipepr-ingress that responded
with `404`, which lead to downtime for all applications within one
minute. Basically the API for the control loop was, that status code
`404` meant "application does not exist" or even better "application was
deleted", so it needed to act on this change and clean up resources.

{{< figure
	src=this_is_fine.jpeg
	alt="This is fine meme"
	caption="This is fine"
>}}

Coming back to the pull request.

What do they want to change exactly?

Here is a cite, that shows what they want to change:

> For instance, a Kubernetes ingress router like
  https://github.com/zalando/skipper must be configured to return
  the status code {503}  instead of {404}, if there is no route for an
  application service endpoint available.

What is `503` and how it is supposed to be used?

`503` is defined in
[rfc9110](https://datatracker.ietf.org/doc/html/rfc9110#section-15.6.4)
and is the status code for StatusServiceUnavailable, which sounds legit at
first. However citing the RFC clarifies the intention of status code `503`:

> The 503 (Service Unavailable) status code indicates that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance, which will likely be alleviated after some delay. The server MAY send a Retry-After header field (Section 10.2.3) to suggest an appropriate amount of time for the client to wait before retrying the request.

If you check when `503` status code is used today in http proxies, you
will find for example circuit breaker, load shedder or in some cases
if the proxy could not reach the backend.  I think this is pretty much
inline with the RFC, for example load shedding and "temporary
overload". However a "route not found" case I don't see
represented. "Route not found" is not temporary, it's the
configuration, that might change after a deployment, but it is not the
regular case.

Let's understand the reasoning of the change proposal:

> In particular, a middleware must not return a status code {404}, if the
  service would have returned a different status code, as this could mislead
  clients into thinking that the resource does not exist when it actually does.

Note: In the case of the incident the proxy did know no backend service.

> Clients can safely assume that {4xx} status codes are always authoritative and
  comply with the end-to-end principle (see <<256>>). While a middleware can
  authenticate a request and return {401} or {403} on behalf of the service, it
  must never return a misleading status code, e.g. {404} or {410}, if the service
  is not configured or not responding.

They say that a client calling some URL should be able to use `404`
status code as the only thing it has to check for "an item does not
exist". So for `https://example.org/items/foo` reponse `404` means foo
item does not exist.  This sounds also legit to me, even if I have a
different opinion, because you can construct also weird examples like
`https://example.org/items/foo/bar/qux` , that does not exist, even if
`/items/foo` would exist. Normally you would get requests to some old
wordpress security vulnerability, so something like
`<some-path>/foo.php`.

What is the definition of `404` status code?

> 15.5.5. 404 Not Found
  The 404 (Not Found) status code indicates that the origin server did not find a current representation for the target resource or is not willing to disclose that one exists. A 404 status code does not indicate whether this lack of representation is temporary or permanent; the 410 (Gone) status code is preferred over 404 if the origin server knows, presumably through some configurable means, that the condition is likely to be permanent.
  A 404 response is heuristically cacheable; i.e., unless otherwise indicated by the method definition or explicit cache controls (see Section 4.2.2 of [CACHING]).

So a middlebox which is a proxy cache can heuristically cache the
`404` response.  There is a status code `410` which should be
preferred for deleted resources. This would be in the case of the
incident a possible way to differentiate between "deleted" and "does
not exist" or "no matching route found".

Why do I think a proxy that has no matching HTTP route for a given
HTTP request should return `404`?

As a proxy to enable fast error detection you should indicate the most
likely happening to guide your automations and investigators.  Given
that availability tools respond with `503`. Given if a backend does not
listen to a port and Linux kernel return connection refused (oom of an
application instance for example), which proxies will turn to
a `502`. Given that a severe error of the proxy or the application will
be a `500`. If you serve public endpoints then you know, if you ever
looked into your access logs that a quite big fraction of traffic are
security scanners, scanning all day for old vulnerabilities. As a
proxy you repond by `404` and it's all good.

Why should we change a `404` to a `5xx`?

The proposal says:

> a middleware must always return a temporary, non-authoritative server-side status code, such as:
  * **{502} — Bad Gateway**,
  * **{503} — Service Unavailable**, or
  * **{504} — Gateway Timeout**.

While the RFC for `404` says:

> A 404 status code does not indicate whether this lack of representation is temporary or permanent

So `404` could be temporary, which does not contradict what the
middleware responds.  Even worse a temporary `404` can not safely
assumed by the client that it is always authoritative.  This means we
have another clear contradiction to the proposed arguments about the
end-to-end semantics.

The proposal also mentions:

> The end-to-end semantics of HTTP status codes is a fundamental part of the API contract.

Given that `404` could be temporary and the RFC shows `410` as a more
clear permanent status code, maybe the guideline should rather clarify
the use of `404`. Even better it could tell about `410` instead.

Why do I think `404` is the right status code for HTTP proxies that do
not find a matching route?

A route can be created, update and deleted by someone any time, so
temporary miss configured route returns `404` and if fixed again it
returns again the result as expected. This seems to be reasonable
based on the RFC.

Another more practical reason is that the amount of crawlers,
vulnerability scanners, `<bot-you-name-it>` scan all the day your web
page every minute my tiny box hosting this blog gets traffic from some
of these and I don't care at all. Sometimes I name this "random
internet noise".  I also stopped logging them, because I don't want to
waste disk for this garbage. Even better our default in our Kubernetes
ingress layer for logs is to disable logs for `2xx`, `3xx`, `404` and
`429`, because it's a waste of disk, money and compute. Even better
less people are distracted because someone started a vulnerability
scanner against a proper zero trust architecture with default
authentication all over the place. In the last 10 years running like
this there was no finding at all.

Here 1d of status codes for my blog.szuecs.net (enabled for one day):

```
$ awk '{print $9}' blog.szuecs.net/blog.szuecs.net_access.log.1 | sort | uniq -c
    401 200
     18 301
      3 304
    165 400
   2226 404  <- 5x more than valid requests
     17 405
      1 408
```

So 5 times more traffic from vulnerability scanners, than from
google-bot, bing-bot, ... and you reading this post!

Another reason why I think it does not even matter to identify an
issue like this via status code is, that with the raise of Opentracing
and the proper standard [Open Telemetry](https://opentelemetry.io/)
(short: OTel), provides a proper tool that easily identifes that the
call chain of a `404` response ended at the HTTP proxy in case you need
to know who responded during a failure analysis.

Even developers that run their workload behind a proxy layer will want
to have a `404` instead of `5xx` response, if you tell them that their
alerting of `5xx` will create false alerts by random internet noise.

/EOF
