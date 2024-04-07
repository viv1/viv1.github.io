---
layout: post
title: Real World Production Issue And Debugging
date: 2024-03-25 18:35
category: Debugging
tags: ["debugging", "production", "saml", "sso", "ntp", "ntpd", "log", "monitoring", "metric"]
description: This is an intriguing story of a real world production issue that we encountered and how we navigated our way to its resolution.
pin: true
---
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [The Incident](#the-incident)
- [Architectural Context](#architectural-context)
- [Initial Analysis](#initial-analysis)
- [Head scratching](#head-scratching)
- [More head scratching and the path to root cause](#more-head-scratching-and-the-path-to-root-cause)
- [The Eureka Moment](#the-eureka-moment)
- [Fix and Learnings](#fix-and-learnings)

<!-- TOC end -->

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

Recently at work I came across an issue, which made me nostalgic, because it reminded me of an interesting production bug that we faced in one of my previous companies a few years ago. Read on for the story...

<!-- TOC --><a href="#" name="the-incident"></a>
## The Incident

One of our services (let's call it `Service SP`) served as a *READ-only portal* for the customers. However, the content was managed by our internal team. One evening, couple of our internal team members (let's call them `Operator A` and `Operator B`) reported that they were not able to login to the service. They said that they were able to login until this morning, but it started failing for them since a few hours ago. The browser would redirect them to a series of redirects and eventually gave them error. They did not even see a login page.

<!-- TOC --><a href="#" name="architectural-context"></a>
## Architectural Context

For context, let me brief you about our architecture for logins.

Our architecture comprised several microservices, including the `Service SP`. Central to our authentication process was a microservice responsible for `Single Sign-On (SSO)` using `Security Assertion Markup Language (SAML)` assertions (let's call it `Identity Provider` or `IdP`). This setup enabled SSO logins across our suite of microservices, streamlining access and enhancing security.

And in case you are not aware, here's a brief about how the `SSO` flow using `SAML` works:

![SAML flow](../assets/img/SAML_flow.webp?version%3D1712507556464)


<!-- TOC --><a href="#" name="initial-analysis"></a>
## Initial Analysis

We quickly confirmed that there was no recent deployment to `Service SP`. Nor was there was any deployment to our `Idp` service.

We checked the operational health of `Service SP` and `Idp Service` compute instances, and they were all fine and running. There was no cpu or memory spikes, all good. The service applications inside the service instance was running as well, reporting no issues.

Both had already tried using incognito window, clearing browser cache, and using different browsers.


<!-- TOC --><a href="#" name="head-scratching"></a>
## Head scratching

And then, another internal team person (`Operator C`) reaches out to us with the same issue.

Fearing a widespread issue, we tried to replicate the issue using our own accounts. 3 of us tried, and it WORKED.... for all of us. While we were happy that this is not a wide spread issue, we were still not certain what's going on.

In between, we had checked *logs*. The request came, it got rejected. The `Idp` was sending back "*Unauthenticated*" error. Unfortunately, our logging had gaps (clearly!) and it did not log the error reason even in the SAML assertion response.

Clearly, there was something specific to their account. But what ? Their `SSO` is completely broken. But how ? Maybe their relevant `SSO` data stored in our `Idp DB` was corrupted ?

Just to be sure, we asked the `operators` to see if their `SSO` worked when they logged in from some other service (say `service SP2`). Because if that also fails, that must be it. We would reset their `SSO` credentials, and things would start working again. And all would be right with the world. We can pack our bags and go home.

Except, `SSO` login worked for all of them when they logged in using `SP2`. So, the `DB` credentials were all fine, and it must be something only related to the affected `SP`. But how ?

<!-- TOC --><a href="#" name="more-head-scratching-and-the-path-to-root-cause"></a>
## More head scratching and the path to root cause

Hmmm.....

Now, `Operator A` reaches out us again saying it started working for him. `Operator A` said they kept on retrying the `SSO` login attempts after clearing cookies. It worked after a few retries. And now, it keeps working all the time.

When we heard this, we kind of figured out what must be going on. The `SP` was running on 3 instances at that point in time.

Our load balancers used `"special" cookie` based sticky sessions. Basically, this cookie is sent to the client's browser on the first response and is expected to be returned by the browser on subsequent requests. This allowed it to ensure that once the request for a client goes to a specific instance, it continues to get served by that same instance.

When `Operator A` retried after clearing their session cookies, the load balancer established a new session, associating it with a different, healthy instance. This change broke the sticky session with the affected instance, allowing them to bypass the issue.

Previously, when they used a different browser, or cleared their cache previously, they were again associated with the affected instance due to luck.

What instance your request is served with the first time, is a matter of luck; and continuing to get served by the same network is a matter of stickiness.

From logs, we realized that two of our instances were actually rejecting `SSO` requests. Only on instance was serving correctly. This must be why the chance of getting attached to an unhealthy instance was much higher.

While we had not figured out the root cause of why the compute instances were "unhealthy", we could now at least mitigate this for all our operators. We simply removed (not terminated, so that we could investigate further) the affected compute instances from the Load Balancer, and attached 2 new ones. Now, `SSO login` was working as before, seamlessly.

<!-- TOC --><a href="#" name="the-eureka-moment"></a>
## The Eureka Moment

With less pressure, we were now trying to understand the actual root cause of why the instances were rejecting the requests. We were trying to find something, or anything from the logs.

For the rejected requests, closely looking at the logs for each request, we see:

```
SP logs:
SAML request generated ...time: 18:05:00PM

Idp logs:
SAML assertion for failure generated.... time: 19:05:05PM
```

Did you notice ? The SP logs are timestamped about ~1 hour in the past.

Once we noticed this, we knew what was wrong. Our SAML request generation keeps the not valid after validity of about 45 mins from now. Since the clock was so out of sync, The `NotOnOrAfter` value was still 15 minutes behind the current actual time.

It was the same in the other instance, in which the system clock was 55 minutes behind. The `ntpd` daemon process, which is responsible for synchronizing system time, had crashed and stopped.

We logged into the instances and quickly checked the system clock timing, and we were correct. So, this was why any requests that went through these instances actually got rejected.


<!-- TOC --><a href="#" name="fix-and-learnings"></a>
## Fix and Learnings

The fix was simply to restart the `ntpd` process again. It took some time, since `ntpd` does not do huge jumps in time. It gradually synchronizes the time. However, the instances were back to the correct system time.

When we resolve an issue, we look for ways to avoid the same in future. We did / planned the following:

- Obviously improved our logging gaps. Had we seen the `NotOnOrAfter` error earlier, we would have found the root cause earlier.
- Improved our monitoring of system clock timings. We added a new host metric to measure the sytem clock timing for our instances. We added a new alarm to our alarming system for when the time drift is higher than 15 minutes.
- We added a cron job to restart the ntpd service in the event it crashed again.


This experience definitely taught as that even minor details, like system clock accuracy, that we take for granted can have major impacts, and how monitoring every aspect of our systems, no matter how small it may seem is important.