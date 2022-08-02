---
layout: post
title:  "type=LoadBalancer Services in Kuryr and cloud provider"
date:   2022-07-26 15:40:52 +0200
tags: openshift openstack kubernetes kuryr loadbalancer octavia ovn
---

`type=LoadBalancer` `Services` are the backbone of many applications based on
Kubernetes. They allow using cloud's load balancer implementation to expose a
`Service` to the outside world. An alternative is using `Ingress`, but that
might feel more complicated, requires running Ingress Controller and only works
with HTTP, so often times you're forced to use the hero of this post. Let's then
dive in into how it works with OpenStack as Kubernetes or OpenShift platform.

## Octavia

First we got to understand how load balancers are done in OpenStack. The
LBaaS project here is called Octavia. By default Octavia uses Amphora provider
(providers are like backends for Octavia). Amphora implements load balancers by
spawning small VMs (called… Amphoras) with HAProxy and a tiny agent on them.
Octavia will then call the agent each time HAProxy configuration needs to be
adjusted. And HAProxy will obviously serve as the LB itself. The idea is simple
but the downside is obvious - VMs consume vCPUs and memory.

A modern way to tackle the problem comes with OVN. If your cloud uses it as a
Neutron backend, you can also use the Octavia OVN provider. Instead of spawning
VMs it will set up load balancers in OVN, greatly reducing the overhead they
create. The downside here is that OVN LBs lack any L7 capabilities and in
general only recently started to catch up with Amphora's features.

## `type=LoadBalancer` with cloud-provider-openstack

Kubernetes needs to do some trickery to actually use cloud's load balancers to
expose `Services`. First of all there's a concept of `NodePort` `Services`,
which expose `Service`'s pods on a random port of the node (from a specified
range). This means that you can call node on a specified port and the traffic
will be redirected to the `Service` pods. This allows cloud provider to create
an LB and add all the K8s nodes as its members. External traffic will reach the
LB, get redirected to one of the nodes and node will direct it into the
`Service` pod. But what if that particular node doesn't have any matching pod?

### ExternalTrafficPolicy

By default in the case described above, the traffic gets redirected to the node
that has a matching pod. This creates a problem - traffic loses the source IP
which may be important for some applications. K8s solves this by introducing
`ExternalTrafficPolicy` setting on `Services`. If you'll set it to `Local`,
you're guaranteed that the traffic will never get redirected. But you cannot
expect all the nodes will have a pod of your `Service`, so how to deal with
that? Kubernetes requires the LB to healtcheck the members of the LB. That way
even with `ETP=Local` you're guaranteed to hit the nodes that actually hold the
correct pods.

In Octavia healthchecks are called health monitors and you can easily
configure your cloud-provider-openstack to create them:

{% highlight ini %}
[LoadBalancer]
use-octavia = True
create-monitor = True
monitor-delay = 3
monitor-max-retries = 3
monitor-timeout = 1
{% endhighlight %}

So what's the problem here? OVN Octavia provider doesn't support health
monitors [until OpenStack
Wallaby](https://docs.openstack.org/releasenotes/ovn-octavia-provider/wallaby.html#new-features).
In case of RH OSP it's until release of version 17.0. This means that
`ETP=Local` `Services` will be unreliable there.

### Other issues

There are other problems related to integration between Octavia and Kubernetes.
[Legacy in-tree cloud provider]({% post_url 2022-07-14-capo-mapo-cloud-provider %}#cloud-provider-openstack-in-openshift)
doesn't support protocols different from TCP. This is because legacy provider
was developed with Neutron LBaaS in mind, which didn't support that
protocol[^1]. Another deficiency here is that in-tree cloud provider won't use
Octavia bulk APIs and anyone who worked with Octavia and Amphora knows that
it's slow to apply changes. This means that if you have a high number of nodes,
creation of `type=LoadBalancer` `Services` will take long because members are
added one-by-one, waiting for the LB to become ACTIVE again. We've got reports
about it taking around and hour for 80 nodes to get added. And note that the
floating IP is added as the last operation on LB creation, so until it finishes
you cannot call your `Service` from the outside. Using OVN Octavia provider
helps quite a bit because it's significantly faster in terms of time to
configure an LB.

Using the modern out-of-tree cloud-provider-openstack helps quite a bit here.
It'll allow you to create UDP and even SCTP LBs and will use bulk APIs to speed
up LB creation. But we've still found
[some](https://bugzilla.redhat.com/show_bug.cgi?id=2042976)
[issues](https://bugzilla.redhat.com/show_bug.cgi?id=2100135) with it. Turns
out it wasn't tested much with OVN Octavia provider and the bulk APIs it is
using weren't functioning correctly in OVN provider. This is fixed now, but you
need to make sure OpenStack cloud you're using is upgraded.

## Kuryr-Kubernetes and LBs

You might know that Kuryr-Kubernetes is a CNI plugin option that ties the
Kubernetes networking very closely to Neutron. In general it will provide
networking by creating a subport in Neutron for every pod, and connect them to
the node's port, being a trunk port. This way isolation is achieved using VLAN
tags. This also means that the Octavia load balancers for `Services` can have
pods added as members directly, without relying on NodePorts. This alone is
solving the issue of `ETP=local` as Kuryr will make sure only correct addresses
are added to the LB, but also greatly speeds up LB creation time because it
doesn't need to add all the nodes to the LB as members. All that's needed to be
done for a `Service` to become `type=LoadBalancer` is to attach a floating ip to
it.

Kuryr is also actively maintained, meaning that it implements newer APIs,
enabling UDP and SCTP load balancing even for older OpenShift versions.

It's worth saying that using Kuryr with OpenShift will greatly increase the
number of LBs created in Octavia, as they're created not only for
`type=LoadBalancer` `Services`, but also the `type=ClusterIP` ones. In bare
OpenShift installation that's around 50 LBs, so Amphora is not recommended as
the provider due to resource consumption. Please also note that there are
scalability issues when a high number of pods is created at once, as it puts a
lot of stress on the Neutron API.

## Summary

I hope this helps to decide which load balancing model is better for your use
case. To overcome the listed Octavia deficiencies you need a version of
OpenShift allowing you to use the external cloud provider and new OpenStack
(even newer if you're using `ETP=Local`. On the other hand using Kuryr will put
a lot stress onto your OpenStack cluster as Neutron will be used not only to
network VMs but also all the pods and load balancers.

[^1]: In fact we're pretty lucky here, legacy cloud provider uses gophercloud
      (OpenStack client for golang) version that doesn't have a notion of 
      Octavia. Only a clever hack and the fact that Octavia v1 API is identical
      with Neutron LBaaS allows it to work with Octavia.
