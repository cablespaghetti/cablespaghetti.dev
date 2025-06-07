---
layout: post
title:  "Using AWS Spot Instances in your production EKS cluster"
description: "How to use AWS Spot instances in your production EKS cluster without causing an outage."
date: 2021-03-05 17:00
tags: kubernetes aws spot
---

At work we have a number of fairly large Kubernetes clusters on Amazon EKS; some with 50 or 60 "xlarge" nodes. This amount of compute on AWS can cost a fortune every month, so of course we wanted to do what we could to reduce this.

## How do Spot Instances work?

Spot Instances are essentially a way for AWS to sell any spare capacity they have, after all the Reserved and On Demand customers have got their instances provisioned. 

This capacity is available without committing for 1 or 3 years like Reserved instances and you get a roughly 70% to 90% discount over the On Demand price (usually about 70% for recent instance types).

The downside of this is that if an On Demand customer comes along and needs the capacity you're using, your instances will be terminated with only 2 minutes warning.

## But I thought they were a bad idea for prod?

Previously the "spot market" operated like an auction; you set the maximum price you were willing to pay and other people could outbid you and take your capacity. However [in 2017 AWS simplified this](https://aws.amazon.com/blogs/compute/new-amazon-ec2-spot-pricing/) to make using Spot Instances less intimidating and unpredictable; now the price is set by AWS rather than demand.

This means that the only way your Spot Instances will be terminated now is if an On Demand customer needs the capacity and AWS doesn't have any other capacity available.

Another change which made Spot Instances more suitable for production was the introduction of [Auto Scaling Group Launch Templates with Mixed Instance Policies](https://aws.amazon.com/blogs/aws/new-ec2-auto-scaling-groups-with-multiple-instance-types-purchase-options/) in 2018 and then [Capacity-Optimized Allocation Strategies](https://aws.amazon.com/about-aws/whats-new/2019/08/new-capacity-optimized-allocation-strategy-for-provisioning-amazon-ec2-spot-instances/) in 2019. The combination of these things made it possible to have a normal Auto Scaling Group request a mixture of instance types with some Spot and some On Demand.

For example you can request m4.xlarge, m5.xlarge and m5a.xlarge with 20% On Demand. You will then get 20% of your requested capacity as On Demand and Spot instances will make up the rest out of the selected types. The instances you get are based on which Amazon has most spare capacity of. This minimises the chance of your instances being terminated and maximises the chance of your ASG being able to provision instances.

In December 2020 [AWS added Spot Instance support to Managed Node Groups](https://aws.amazon.com/blogs/containers/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups/). This takes a lot of the complexity out of managing your nodes, but doesn't change much of this information. I have noted where things differ between the two approaches.

## When to use Spot Instances

Spot is not for everyone; it requires your workloads to handle individual Kubernetes Pods being terminated without causing downtime. This is of course good practice anyway, but most companies have at least a few legacy apps which can only run a single instance; these are not suited to running on spot. However I'll talk about how you and "pin" these apps to your On-Demand instances later on.

Disclaimers out of the way, we went primarily with Spot for a few reasons. Firstly, that it is the cheapest possible way to get compute on AWS; even if you pay for a Reserved Instance for 3 years upfront it's about 60% savings (depending on region and instance type) and the more sane 1 year term is about 40%.With Spot Instances you can typically expect about a 70% saving.

Secondly, our load is quite spiky; at peak load we need about 80 instances in our main production cluster but the rest of the time it's maybe only 40 instances; so reserved instances would only make sense for about half the cluster anyway.

## Preparation

Even with the perfect workload, it still takes a bit of work to get your cluster ready for running on Spot instances.

### Pod Topology Spread Constraints 

If your Kubernetes cluster nodes are fairly likely to disappear, it's a good idea not to run all the instances of a particular service on a single Node. The old way of doing this before Kubernetes v1.19 was using [Inter-Pod Affinity Rules](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity). However now with v1.19 available on EKS you can use [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/) which are a bit more flexible and easy to configure.

### AWS Node Termination Handler

If you are _not_ using Managed Node Groups, the [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler) listens for "Spot Instance Termination Notification" events from AWS (and a number of others). These events come 2 minutes before your nodes are terminated.

When it receives one of these events it will gracefully Cordon and [Drain the Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/), to move the Pods to other instances before the Node is terminated.

One of the advantages of Managed Node Groups is that this is all handled for you without running additional services.

## Setting up your Auto Scaling Groups

The Kubernetes [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler) is quite particular about how your Auto Scaling Groups (or Managed Node Groups) are set up. This will also impact how your Nodes are labelled when they join the cluster, so you can identify which ones are On Demand and which are Spot.

The Cluster Autoscaler only works properly if you have a separate Auto Scaling Group (or Managed Node Group) for each Availability Zone you want to use. It also needs the Nodes to be of the same *CPU and RAM capacity*, so it can properly estimate what will fit on new nodes it spins up.

In practice we've gone with this configuration:

* A set of three Auto Scaling Groups set to request `m5a.xlarge` On Demand instances; for `us-east-1a`, `us-east-1b` and `us-east-1c`. These Nodes are labelled with `capacity-type=on-demand` when they join the cluster by adding parameters to the `bootstrap.sh` script in the EKS Worker Node AMI; [this article covers how that works](https://aws.amazon.com/blogs/opensource/improvements-eks-worker-node-provisioning/).
* A set of three Auto Scaling Groups using a mixed instances policy (mentioned above) which request Spot Instances of `m5.xlarge`, `m4.xlarge` and `m5a.xlarge` in the same AZs. These are labelled with `capacity-type=spot`.
* A set of three Auto Scaling Groups requesting Spot `m6g.xlarge` ARM instances ([see my ARM cluster article]({% link _posts/2021-02-20-managing-multi-arch-kubernetes-clusters.markdown %})).

The only difference when using [Manged Node Groups](https://aws.amazon.com/blogs/containers/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups/), is that changing the Node labels can be done from the AWS Console rather than editing the `bootstrap.sh` parameters.

We then use the [Priority Expander](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/expander/priority/readme.md) in the Kubernetes Cluster Autoscaler to make the Spot Auto Scaling Groups higher priority than the On Demand ones, so the On Demand groups are only scaled up if capacity is not available in the Spot groups. The [Helm Chart](https://github.com/kubernetes/autoscaler/blob/master/charts/cluster-autoscaler/values.yaml) helpfully makes this really easy to configure.

With Managed Node Groups the cluster autoscaler only knows about ASG names not the Managed Node Group names. There is an option issues on the AWS [containers-roadmap GitHub](https://github.com/aws/containers-roadmap/issues/1304) to make the ASG names more identifiable. Until then I've written [a little utility to work around it](https://github.com/cablespaghetti/priority-expander-eks-managed-nodegroup-configurer). There is also [this open issue](https://github.com/kubernetes/autoscaler/issues/3956) for the v1.19 cluster autoscaler which is worth watching out for.

## Node Affinity for apps we want to run on On Demand

Because we've split up the Spot and On Demand Instance Auto Scaling Groups and used that to label the Nodes differently in the cluster, we are now able to use those labels to schedule certain Pods to our On Demand instances. This is very useful for those Legacy apps which will cause downtime if restarted. [This documentation covers how Node Affinity works](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/).

We also set give these Pods a high priority [through PriorityClasses](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/) to minimise the chance of them being evicted from their Node.

## Final Notes

I hope this gives you a good idea of how to use Spot instances in your EKS clusters. I expect most of it could be applied to a non-EKS cluster in AWS, and I plan on trying this with [k3s](https://k3s.io/) in the near future.

I'm always happy to chat to people working on similar challenges, so get in touch on [Twitter](https://twitter.com/cablespaghetti) if you have any questions or just fancy a chat!
