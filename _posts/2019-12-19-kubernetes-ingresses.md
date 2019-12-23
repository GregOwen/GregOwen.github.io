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

I recently had to configure a new Kubernetes ingress at work. Before I deployed the ingress into our network, I wanted to understand what it was and what it would do.

There are [lots](https://blog.getambassador.io/kubernetes-ingress-nodeport-load-balancers-and-ingress-controllers-6e29f1c44f2d) of [blog](https://codeburst.io/kubernetes-ingress-simply-visually-explained-d9cad44e4419) [posts](https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-ingress-guide-nginx-example.html) out there explaining Kubernetes ingresses, but all the ones I found were too abstract for what I needed. These posts covered why you would want an ingress object and what ingress objects do at a "software architect drawing on a whiteboard" level, but none of them got to the level of detail that I needed as a practitioner. Unfortunately, the [Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/) [docs](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) weren't much help either. None of these explanations gave me the intuition I needed to be confident installing an ingress into our network.

Specifically, I wanted to know the following:

1. What resources will be created in my cloud provider when I create an ingress object?
1. How does my traffic flow through an ingress?
1. What is an ingress controller and how does it relate to an ingress?

In this post, I'll answer those questions and hopefully build up an intuition for what in ingress is. This post will NOT tell you why you would want to use an ingress or give you a nice "boxes and arrows" diagram explaining where an ingress would sit in your network. The market for that kind of post is totally saturated, so I'll just direct you to check out any of the links a few paragraphs up.

# What is an ingress?

An ingress is a piece of configuration for an ingress controller.

There are a few important pieces to this:
1. Creating an ingress may or may not provision new virtual machines or load balancers in your cloud provider, depending on which ingress controller you're using[^ingress-creation-resources].
1. Creating an ingress does nothing if you don't have an ingress controller[^gce-ingress-controller].
1. If you have an ingress controller, each ingress defines a part of the ingress controller's configuration. The total configuration for the ingress controller is the result of merging all of the little bits of configuration from your different ingresses.

Basically, ingresses allow you to distribute your ingress controller's configuration across multiple Kubernetes resources.

# Ok, so what's an ingress controller?

The ingress controller is the actual running code that provides the configurable routing features all the blog posts talk about. Some ingress controllers will create new load balancers for each ingress object, but some of them are themselves load balancers that get their configuration from ingress objects rather than a static file.

For the rest of this blog post, I'll discuss the nginx ingress controller since it's the only multi-cloud controller that's officially part of the Kubernetes project[^only-multicloud]. The nginx ingress controller is the "single load balancer that gets configured by ingress objects" type of ingress controllers rather than the "thing that launches many load balancers" type.

If we look at the [nginx ingress controller installation guide](https://Kubernetes.github.io/ingress-nginx/deploy/), we see that it asks us to `kubectl apply -f` two files: a [deployment](https://raw.githubusercontent.com/Kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml) and a [service](https://raw.githubusercontent.com/Kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml) to access that deployment. The pod created by the deployment listens for changes to ingress objects in your Kubernetes cluster and runs an nginx load balancer whose rules are configured by those ingress objects. The service allows other Kubernetes pods (and possibly the wider internet) to talk to the load balancer on the pod.

Basically, using ingresses and an nginx ingress controller is just a way to run an nginx load balancer in your Kubernetes cluster and configure it using a bunch of small pieces of configuration rather than modifying a single `nginx.conf` file.

# Why would I want to use ingresses?

Why would you want to split your networking configuration up into multiple ingress objects? Well, if you have a lot of engineers working on a lot of different services, keeping all of your network configuration in one file starts to become painful. It's much nicer to define each service's networking configuration in a place that's specific to that service, for a few reasons:

1. The configuration can be "close" to the service definition (the files can be in the same directory and the Kubernetes resources in the same namespace). This makes the configuration easier to discover, and it also makes it easier to control which engineers can change your service's networking: your continuous integration tool can use an ["owners" file](https://docs.gitlab.com/ee/user/project/code_owners.html) to guarantee that pull requests that change things in a service's directory must be approved by that service's team, and Kubernetes allows you to enforce that resources in a service's [namespace](https://Kubernetes.io/docs/reference/access-authn-authz/authorization/#review-your-request-attributes) can only be changed by that service's team.
1. An engineer reading the configuration can see exactly the information relevant to the service and nothing else.
1. An engineer writing the configuration doesn't have to contend with merge conflicts from all of the other engineers who want to change the configuration for their own services[^similar-to-bazel].

# How does traffic flow through an ingress?

Let's see what happens when we create a new ingress object. I've already deployed an nginx ingress controller following the setup instructions [here](https://Kubernetes.github.io/ingress-nginx/deploy/). We can see that we've got a one-pod deployment running the nginx ingress controller and a service in front of that pod:

```
$ kubectl --namespace=ingress-nginx get pods
NAME	  					READY	STATUS	RESTARTS	AGE
nginx-ingress-controller-6bf4985c6d-szz9r	1/1	Running	0		1h

$ kubectl --namespace=ingress-nginx get svc
NAME	  	TYPE		CLUSTER-IP	EXTERNAL-IP		PORT(S)		AGE
ingress-nginx	LoadBalancer	10.47.254.64	<NGINX_SVC_EXT_IP>	80/TCP,443/TCP	1h
```

(I've redacted the public IP address of the ingress controller service, but the `EXTERNAL-IP` field in that last row is non-empty).

I've also deployed a super simple webapp in a two-pod deployment called `webapp` and put an internal-facing service called `webapp-svc` in front of it:

```
$ kubectl --namespace=webapp get pods -o wide
NAME			READY	STATUS	RESTARTS	AGE	IP
webapp-6f764c447c-bvgmj	1/1	Running	0		1h	10.44.2.242
webapp-6f764c447c-w7j25	1/1	Running	0		1h	10.44.7.250

$ kubectl --namespace=webapp get svc
NAME	   	TYPE		CLUSTER-IP	EXTERNAL-IP	PORT(S)	AGE
webapp-svc	ClusterIP	10.47.252.244	<none>		443/TCP	1h
```

Here's the ingress spec I'm going to deploy.

```
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: webapp-ingress
  namespace: webapp
  annotations:
    Kubernetes.io/ingress.class: "nginx"
  spec:
    backend:
      serviceName: webapp-svc
      servicePort: 443
```

There are three important pieces here:

1. I'm launching this ingress in the same namespace as my webapp service. Ingresses [cannot direct traffic to services in other namespaces](https://github.com/Kubernetes/Kubernetes/issues/17088) without using some pretty hacky workarounds. Note that my ingress *controller* is in a separate namespace (`ingress-nginx`), but the *ingress* is in the same namespace as my backend service (`webapp`).
1. I'm marking this ingress with `Kubernetes.io/ingress.class: "nginx"` so that only the nginx ingress controller will use this ingress as part of its configuration. I have to do this because I'm running on GKE and I want to make sure that the global GCE ingress controller doesn't try to track this ingress.
1. This ingress will forward all traffic to port 443 on the service in front of my webapp.

When I deploy the ingress, I see this:
```
$ kubectl --namespace=webapp get ing
NAME	  	HOSTS	ADDRESS			PORTS	AGE
webapp-ingress	*	<NGINX_SVC_EXT_IP>	80, 443	10m
```

Note that the IP address is the same as the IP address of the nginx ingress controller service. This is because the nginx ingress controller is running an nginx load balancer and this ingress is just configuring that load balancer.

Now let's send some traffic to the ingress's IP address:

```
$ curl -k "https://<NGINX_SVC_EXT_IP>/test"
```

If we look at the logs on the ingress controller, we can see what happened:

```
$ kubectl --namespace=ingress-nginx logs nginx-ingress-controller-6bf4985c6d-szz9r | grep "GET /test"
... [22/Dec/2019:23:23:23 +0000] "GET /test HTTP/1.1" ... [webapp-webapp-svc-443] [] 10.44.2.242:9001 ...
```

I've removed some unimportant details from the log line. The important parts that remain show us that my request (`"GET /test HTTP/1.1"`) hit the nginx ingress controller pod, was forwarded to my webapp service (`[webapp-webapp-svc-443]` - according to the nginx log format, this is the `$proxy_upstream_name` variable, which seems to include the namespace, service name, and port) and then handled by one of the webapp pods (`10.44.2.242:9001` - this is the IP address of the pod `webapp-6f764c447c-bvgmj`).

So at least for the nginx ingress controller, the traffic flow looks like `client` --> `nginx ingress controller service` --> `nginx ingress controller pod` --> `webapp service` --> `webapp pod`.

For the ingress controllers that launch new load balancers for their ingresses, traffic presumably flows through those load balancers rather than through the ingress controller itself.

# What happens when I create an ingress?

According to the [nginx ingress controller docs](https://Kubernetes.github.io/ingress-nginx/how-it-works/#nginx-model), the nginx ingress controller listens for changes to ingress objects. Whenever an ingress object changes, the controller translates the new state of the world into an nginx conf and compares it to the nginx conf that the controller is currently running. If the new conf is different from the old conf, the controller tells the nginx load balancer to load the new conf.

Practically speaking, this means that you can make sure that your changes have actually taken affect by checking the nginx ingress controller's nginx.conf:

```
$ kubectl --namespace=ingress-nginx exec -ti $CONTROLLER_POD_NAME -- cat /etc/nginx/nginx.conf
```

# Conclusion

The actual meaning of an ingress object can be substantially different for different ingress controllers.

With some controllers like the GCE ingress controller or the AWS ALB ingress controller, each ingress creates a new load balancer that implements the routing rules specified by the ingress. With these controllers, traffic directed at the ingress's IP address flows through the ingress-specific load balancer to the backend service(s) configured in the ingress.

For other ingress controllers like the nginx ingress controller, an ingress is just a piece of distributed configuration for a single global load balancer. With these controllers, traffic directed at the ingress's IP address flows through the controller itself, which then forwards traffic to the backend service(s) configured in the ingress(es).

While this flexibility is probably great for people who implement ingress controllers, it makes it hard to develop an intuitive understanding of what an ingress is (this may explain why I couldn't find any documentation that provided that kind of understanding). It also makes it hard to apply intuition gained by working with one ingress controller to a system that uses a different ingress controller: the costs associated with launching a new ingress may be substantially different.

My takeaways from this are:
1. In every ingress object, use the `Kubernetes.io/ingress.class` annotation to be explicit about which type of ingress controller will handle your ingress. This makes it clear what the expected behavior will be and protects against potential regressions if somebody deploys a new type of ingress controller into your cluster.
1. To understand how your ingress will behave, check your ingress controller's documentation rather than the general Kubernetes ingress controller documentation.
1. If you're running an open source project and want to allow third parties to provide different implementations of a particular interface, there's a tradeoff between allowing flexibility to implementers and making the resulting functionality easy to understand for your users. A high-level, non-prescriptive interface means that implementers can develop a wider variety of solutions, but that wider variety of solutions means that the behavior of the interface can be less intuitive to users. I would have found it really helpful if the Kubernetes docs had been more explicit that different ingress controllers could have such different behaviors in terms of how many cloud resources they launched and had suggested that I read up on the specific behavior of the ingress controller I was using rather than trying to understand ingress controllers in general.

[^closeness-and-acls]: This also has implications for access control, since Kubernetes allows you to require different permissions for different namespaces.

[^gce-ingress-controller]: If you're on GCE or GKE, you'll always have the global GCE ingress controller (you don't have to launch it yourself).

[^ingress-creation-resources]: For example, the [GCE ingress controller](https://cloud.google.com/Kubernetes-engine/docs/how-to/load-balance-ingress) and [AWS ALB ingress controller](https://aws.amazon.com/blogs/apn/coreos-and-ticketmaster-collaborate-to-bring-aws-application-load-balancer-support-to-Kubernetes/) will provision a new load balancer for each ingress, but the [nginx ingress controller](https://Kubernetes.github.io/ingress-nginx/how-it-works/) will not.

[^only-multicloud]: From [the Kubernetes docs](https://Kubernetes.io/docs/concepts/services-networking/ingress-controllers/): "Kubernetes as a project currently supports and maintains GCE and nginx controllers." My guess is that that decision went something like this:

    1. Google wanted to make sure the GCE ingress controller was "official" and easy to use
    1. Google didn't want the GCE ingress controller to be the only officially supported ingress controller, since that might look anticompetitive or give the impression that Kubernetes was a GCE-only thing
    1. The Kubernetes project didn't want to have to support every ingress controller under the sun, since that's a huge maintenance burden for something that's not really a core part of the Kubernetes value proposition
    1. Compromise: decide to support exactly one third-party ingress controller from an established company that doesn't run its own cloud. All other controllers get listed as "Additional controllers" that the Kubernetes team doesn't have to maintain.

[^similar-to-bazel]: There are very similar benefits to using Bazel's distributed BUILD files rather than having a single central build file. I used to work at a company that switched to Bazel from SBT, which defines all of its build rules a single centralized `build.sbt` file. By the time we made the switch, that `build.sbt` file had grown so large and convoluted that trying to open it would crash IntelliJ. Distributed configuration, whether for networking rules or build rules, scales a lot better than centralized configuration.