---
title: "What is a Kubernetes ingress, actually?"
date: 2019-12-19T22:00:00-08:00
editors:
classes: wide
categories:
  - programming
tags:
  - kubernetes
  - ingress
  - microservices
  - services
  - service oriented architecture
---

I recently had to configure a new Kubernetes ingress at work.

There are lots of blog posts out there explaining kubernetes ingresses, but all the ones I found were at the same level, and that level was too abstract for what I needed. These posts covered why you would want an ingress object and what ingress objects do at a "software architect drawing on a whiteboard" level, but none of them got to the level of detail that I needed as a practitioner. None of them gave me the intuition I needed to be confident installing one into our network.

Specifically, I wanted to know the following:

1. What resources will be created in my cloud provider when I create an ingress object?
1. How does my traffic flow through an ingress?
1. What is an ingress controller and how does it relate to an ingress?

As a warning, this post will NOT tell you why you would want to use an ingress or give you a nice "boxes and arrows" diagram explaining where an ingress would sit in your network. The market for that kind of post is totally saturated, so I'll just direct you to check out any of the links a few paragraphs up.

# What is an ingress?

An ingress is a piece of configuration for an ingress controller.

There are a few important pieces to this:
1. Creating an ingress does not provision any new virtual machines or load balancers in your cloud provider[^gce-exception].
1. Creating an ingress does nothing if you don't have an ingress controller.
1. If you have an ingress controller, each ingress defines a part of the ingress controller's configuration. The total configuration for the ingress controller is the result of merging all of the little bits of configuration from your different ingresses.

Basically, ingresses allow you to distribute your ingress controller's configuration across multiple kubernetes resources. Why would you want this? Well, if you have a lot of engineers working on a lot of different services, keeping all of your network configuration in one place starts to become painful. It's much nicer to define each service's networking configuration in a place that's specific to that service. That way, the configuration can be "close" to the service definition (the files can be in the same directory and the kubernetes resources in the same namespace), an engineer reading the configuration can see exactly the information relevant to the service and nothing else, and an engineer writing the configuration doesn't have to contend with merge conflicts from all of the other engineers who want to change the configuration for their services[^similar-to-bazel].

# Ok, so what's an ingress controller?

The ingress controller is the actual infrastructure that provides the nice configurable routing features all the blog posts talk about. Essentially, it's a traditional load balancer.

For the rest of this blog post, I'll discuss the nginx ingress controller since it's the only multi-cloud, non-magical controller that's officially part of the kubernetes project.

If we look at the nginx ingress controller installation guide, we see that it asks us to `kubectl apply -f` two files. The core of the ingress controller is a deployment and a service to access that deployment. The pod created by the deployment listens for changes to ingress objects in your kubernetes cluster and runs an nginx load balancer whose rules are configured by those ingress objects. The service allows other kubernetes pods (and possibly the wider internet) to talk to the load balancer.

Basically, using ingresses and an nginx ingress controller is just a way to run an nginx load balancer in your kubernetes cluster and configure it using a bunch of small pieces of configuration rather than modifying a single `nginx.conf` file.

# How does traffic flow through an ingress?

Let's see what happens when we create a new ingress object. I've already deployed an nginx ingress controller following the setup instructions here. I've also deployed a super simple webapp in a deployment called `` and put a service called `` in front of it.

Here's the ingress spec I'm going to deploy.

```ingress spec```

The important part is that this ingress will target the service in front of my webapp. When I deploy the ingress, I see this:

After a little while, the ingress controller finds the new ingress:

Notice that the IP address is the same as the IP address of the ingress controller.

Now let's send some traffic to the ingress's IP address:

```curl command```

If we look at the logs on the ingress controller, we can see what happened:

```ingress controller logs```

We see that my curl command hit the ingress controller's service, then was processed by the ingress controller pod, which forwarded it to one of the webapp service's pods.

```arrows-and-boxes diagram```

TODO: traceroute?

# What happens when I create an ingress?

So if deploying an ingress object doesn't actually create any new infrastructure in my cloud provider account, what does it do?

For that, we can take a look at the ingress controller code TODO: link. The nginx ingress controller polls ....

TODO: what about other resource types?

[^gce-exception] Unless you're on GCE or GKE, in which case your global GCE ingress controller (which you did not have to launch yourself) will provision stuff for you. TODO(greg): check this.

[^similar-to-bazel] There are very similar benefits to using Bazel's distributed BUILD files rather than having a single central build file. I worked at a company that used to use SBT, which defines all of its build rules a single centralized `build.sbt` file. Before we switched to Bazel, that `build.sbt` file  had grown large and convoluted enough that trying to edit it would crash IntelliJ. Distributed configuration, whether for networking rules or build rules, scales a lot better than centralized configuration.