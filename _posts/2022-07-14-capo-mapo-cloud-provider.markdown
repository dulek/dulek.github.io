---
layout: post
title:  "CCM, cloud provider, CAPO, MAPO - OpenStack in OpenShift explained"
date:   2022-07-14 15:40:52 +0200
tags: openshift openstack kubernetes capo mapo
---
Integrating an app with a cloud is a difficult thing in general, but when the app is called OpenShift and implements a whole container platform abstracting the underlying clouds, the task becomes a serious challenge. That's why when researching the topic you might feel lost hearing acronyms like _CAPO_, _MAPO_ or _OCCM_. This blog post's goal is to explain roles of these integration points between OpenShift and OpenStack, the components implementing that integration and why this ended up so complicated. For a more detailed discussion of the integration you can check out [the talk by my colleague Matt Booth presented at OpenInfra Summit in Berlin.](https://www.youtube.com/watch?v=ue0JE4SewCY)

## Integration components

First of all it's important to understand the relationship between vanilla Kubernetes and OpenShift. In general OpenShift uses upstream Kubernetes code, but may carry additional patches or even completely diverge from upstream. Later in this article we'll see that this has an undesirable cost, but obviously it happens sometimes.

In general Kubernetes has two integration components with the underlying cloud - _cloud provider_ and _cluster API provider_. As a rule of thumb you can assume that cloud provider serves the K8s cluster users, while CAPO is what cluster admin interacts with.

### Cloud provider

The cloud provider is responsible for serving the workloads running on the Kubernetes platform. For example, when you're creating a ``Service`` of ``type=LoadBalancer`` then the cloud provider will make sure to create a load balancer on the cloud that will serve traffic for that Service. In the  case of OpenStack, it also manages Node IP addresses, provides an Ingress controller based on Octavia and hosts Manilla and Cinder CSI plugins.

The cloud provider implementation for OpenStack clouds is called [cloud-provider-openstack](https://github.com/kubernetes/cloud-provider-openstack).

### Cluster API provider

The _[cluster API](https://cluster-api.sigs.k8s.io/)_ is a Kubernetes extension that allows Kubernetes to manage its own cluster lifecycle. The _cluster API providers_ are what implement that API. It means it deploys, monitors and removes VMs on which Kubernetes nodes are running. Cluster API provider acts for example when you scale down your Kubernetes cluster. The Cluster API Provider implementation for OpenStack clouds is called [cluster-api-provider-openstack](https://github.com/kubernetes-sigs/cluster-api-provider-openstack) or CAPO.

## cloud-provider-openstack in OpenShift

So far, so good, right? Well, it's a bit more complicated when it comes to OpenShift. Cloud providers weren't always in neat, separate repos. In the past Kubernetes kept them in the main repo and today these are called _in-tree cloud providers_. Up to 4.11 OpenShift only supported the _[in-tree OpenStack cloud provider](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/legacy-cloud-providers/openstack)_. While these are still supported, the Kubernetes community is no longer  allowing new changes to these legacy _in-tree_ cloud providers. Instead, all new feature development is taking place in the _new_ cloud providers, which can be called either _external_, _out-of-tree_ or _CCM_, for _Cloud Controller Manager._ The OpenStack external cloud provider is only supported in tech preview in OpenShift 4.11 and is planned to GA in a future release of OpenShift.

In practice this means that OpenShift's cloud-provider-openstack has several limitations compared to Kubernetes running with the external cloud provider, e.g. doesn't support UDP when it comes to ``Services`` of ``type=LoadBalancer``.

Why have we ended up here, using legacy for so long? Well, making a switch is not trivial. External cloud provider OpenStack has a bit different configuration options (a problem when considering upgrades), made some breaking changes (e.g. regarding node IPs management) and uses different Octavia calls that aren't fully tested with OVN Octavia provider. But hey, at least OpenShift's version of the legacy OpenStack cloud provider had not diverged too much from upstream, so making the switch is possible in general. The same cannot be said of Cluster API provider. 

## CAPO, MAPO and how we got there

In the case of CAPO, the OpenShift version evolved independently from the upstream version. While this allowed OpenShift to move forward more quickly, it also means that API design of the Machine and MachineSet objects diverged from what's in the upstream Kubernetes. Such a situation is hard to maintain as any bugfix made upstream most likely needs to be adapted to a different API on the OpenShift side. The team decided to act to solve the issue.

What we needed was basically a translation layer between OpenShift CAPO and the upstream Kubernetes version of it. We've called it MAPO - _machine-api-provider-openstack_. MAPO is built in a pretty clever way - it watches for the OpenShift version of Machine objects, translates them into Kubernetes Machine objects and uses them to execute upstream CAPO functions that it is vendoring. An alternative approach could just be to run upstream CAPO normally and let MAPO create the K8s version of Machine objects from the OpenShift ones, but that could lead to users being confused by seeing two versions of Machine objects.

It's important to say that MAPO is deployed by default starting from OpenShift 4.11 and currently it's the only supported option. For the users nothing important should change as it is supporting all the APIs that diverged OpenShift CAPO was.

## What's next in OpenShift on OpenStack?

So what's going to happen in these topics in the near future? In the case of MAPO the answer is pretty simple - we'll just make sure to keep the vendored CAPO up to date and implement any new upstream API in our translation layer.

The situation with switching out of legacy cloud provider is more complicated. The OCCM support was initially planned to be completed in 4.11, but we've hit unforseen upgrade issues. The upgrade is complicated as we cannot have two controllers running concurrently and processing events as we would end up with duplicated Octavia load-balancers or see [weird behaviors of Node objects](https://github.com/kubernetes/kubernetes/issues/109793). At the moment the migration path is planned to be completed and GA in one of the upcoming OpenShift releases.

