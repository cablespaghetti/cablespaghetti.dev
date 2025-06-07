---
layout: post
title:  "EKS Managed Node Groups, the good, the bad and the config"
description: "The pros and cons, and how to migrate to Amazon EKS Managed Node Groups."
date: 2021-03-20 17:00
tags: kubernetes aws eks
---

Amazon EKS launched in 2018 to the relief of many who had been managing their own Kubernetes clusters on AWS. However it wasn't as fully featured as some had hoped out of the gate. One of the big improvements Amazon made was to release [Managed Node Groups](https://aws.amazon.com/blogs/containers/eks-managed-node-groups/) in 2019; this removed the need for people to manage their own Auto Scaling Groups and tasks like replacing nodes to upgrade to a new AMI version no longer required a long drawn out manual process or [home grown automation](https://github.com/hellofresh/eks-rolling-update).

Although they were a big step forward for EKS usability, the initial release of Managed Node Groups had some limitations which meant it wasn't suitable for everyone. Most importantly for us, it only supported On Demand instances, but users also couldn't customise the Launch Template for the nodes; this restricted us to using the official EKS Optimized AMI and only customising node labels rather than having full control over the [bootstrap.sh script](https://aws.amazon.com/blogs/opensource/improvements-eks-worker-node-provisioning/).

[Launch template support](https://aws.amazon.com/blogs/containers/introducing-launch-template-and-custom-ami-support-in-amazon-eks-managed-node-groups/) was released in August 2020 and then in December so was [spot instance support](https://aws.amazon.com/blogs/containers/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups/) ([although I only found out a couple of weeks ago](https://twitter.com/rothgar/status/1368457026175602693)). These two features ultimately made Managed Node Groups flexible enough for most users, even awkward ones like me.

You can read a little more about how we've got things set up in my [previous post about spot instances](https://cablespaghetti.dev/2021/03/05/aws-spot-instances-in-production/). However I'll go into more detail on the Managed Node Group specifics in this post. The short version is that we obviously needed Spot support and to customise the taints on our [ARM nodes](https://cablespaghetti.dev/2021/02/20/managing-multi-arch-kubernetes-clusters/).

## The Good

The primary reason we desperately wanted to move to Managed Node Groups was the amount of time and effort it took us to replace our nodes to upgrade to a new AMI. We've been using [hellofresh/eks-rolling-update](https://github.com/hellofresh/eks-rolling-update) for this which is a great tool, but needs to be run manually; this isn't too much of a problem with one or two clusters but for fifteen it gets *very* time consuming.

Whilst it didn't really benefit us as established EKS users, it shouldn't be understated just *how much easier* it is to get started with EKS than it was previously, thanks to Managed Node Groups.

## The Bad

We have a fairly unusual set up, where we have multiple groups of nodes with different priorities; when we scale up we usually get Spot Instances, with On Demand instances only getting launched if there is no Spot capacity available. Unfortunately for this use case the naming of the Auto Scaling Groups which Managed Node Groups create under the hood is a problem; this is because the Kubernetes Cluster Autoscaler doesn't see the nice descriptive name you give your Node Group, it only sees the UUID style hash given to the ASG. I had to [build a tool](https://github.com/cablespaghetti/priority-expander-eks-managed-nodegroup-configurer) to generate Priority Expander configuration for the autoscaler to work around this. However if you're not using the Priority Expander it's not going to be an issue.

## The Config

We found launching managed node groups with our own custom Launch Template to be a little *nuanced*; for example we had some cryptic error messages from Terraform when trying to use a custom user-data script without explicitly setting the AMI. Here's the Terraform we used in case it helps others who need to tweak things in more complex ways:

`user-data.sh`:
```sh
#!/bin/bash -xe
# Inject imageGCHighThresholdPercent value unless it has already been set.
if ! grep -q imageGCHighThresholdPercent /etc/kubernetes/kubelet/kubelet-config.json;
then
    sed -i '/"apiVersion*/a \ \ "imageGCHighThresholdPercent": 70,' /etc/kubernetes/kubelet/kubelet-config.json
fi

# Inject imageGCLowThresholdPercent value unless it has already been set.
if ! grep -q imageGCLowThresholdPercent /etc/kubernetes/kubelet/kubelet-config.json;
then
    sed -i '/"imageGCHigh*/a \ \ "imageGCLowThresholdPercent": 50,' /etc/kubernetes/kubelet/kubelet-config.json
fi

/etc/eks/bootstrap.sh --b64-cluster-ca ${cluster_auth_base64} \
--apiserver-endpoint ${endpoint} \
--kubelet-extra-args '--node-labels=lifecycle="${node_lifecycle}" --register-with-taints="${node_taint}"' \
${cluster_name}
```

We then configure the `template_file` for each set of Node Groups:
```terraform
data "template_file" "userdata_spot_arm64" {
  template = file("${path.module}/user-data.sh")
  vars = {
    cluster_name        = var.cluster-name
    endpoint            = aws_eks_cluster.eks-cluster.endpoint
    cluster_auth_base64 = aws_eks_cluster.eks-cluster.certificate_authority[0].data
    node_lifecycle      = "spot"
    node_taint          = "arch=arm64:NoSchedule"
  }
}
```

We also need to get the AMI from Amazon's public Systems Manager paths:
```
data "aws_ssm_parameter" "eks-worker-ami" {
  name = "/aws/service/eks/optimized-ami/1.19/amazon-linux-2/recommended/image_id"
}

data "aws_ssm_parameter" "eks-worker-ami-arm64" {
  name = "/aws/service/eks/optimized-ami/1.19/amazon-linux-2-arm64/recommended/image_id"
}
```

The there's the `aws_launch_template`:
```terraform
resource "aws_launch_template" "eks-cluster-node-group-worker-nodes-spot-arm64" {
  image_id               = data.aws_ssm_parameter.eks-worker-ami-arm64.value
  name                   = "${var.cluster-name}-eks-cluster-node-group-worker-nodes-spot-arm64"
  vpc_security_group_ids = [aws_security_group.eks-cluster-worker-nodes.id]
  user_data              = base64encode(data.template_file.userdata_spot_arm64.rendered)
  ebs_optimized          = true

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size = 50
      volume_type = "gp3"
      iops        = 3000
    }
  }
  lifecycle {
    create_before_destroy = true
  }
  depends_on = [aws_eks_cluster.eks-cluster]
}
```

Finally the `aws_eks_node_group`:
```terraform
resource "aws_eks_node_group" "eks-worker-nodes-spot-arm64" {
  count           = length(local.subnet-ids)
  cluster_name    = aws_eks_cluster.eks-cluster.name
  capacity_type   = "SPOT"
  node_group_name = "${var.cluster-name}-${count.index}-spot-arm64"
  node_role_arn   = aws_iam_role.eks-cluster-worker-nodes.arn
  subnet_ids      = [element(local.subnet-ids, count.index)]
  instance_types  = ["m6g.2xlarge"]

  scaling_config {
    desired_size = var.spot-arm64-min-hosts-per-az
    max_size     = var.spot-arm64-max-hosts-per-az
    min_size     = var.spot-arm64-min-hosts-per-az
  }

  lifecycle {
    ignore_changes = [scaling_config.0.desired_size]
  }

  launch_template {
    id      = aws_launch_template.eks-cluster-node-group-worker-nodes-spot-arm64.id
    version = aws_launch_template.eks-cluster-node-group-worker-nodes-spot-arm64.latest_version
  }

  tags = {
    Name        = "${var.cluster-name}-${count.index}-spot-arm64"
    Environment = var.cluster-name
  }
}
```

## The Migration Steps

After migrating to Managed Node Groups, life obviously becomes much easier with managing nodes and Kubernetes upgrades. However the process to migrate from our old Auto Scaling Groups took a bit of thought to get right. So here it is:

1. We were upgrading to Kubernetes 1.19 at the same time, so we followed the normal steps to do that. Upgrading the EKS Cluster control plane itself and ensuring kube-proxy, CoreDNS, the Cluster Autoscaler etc were up to date for that version, as per the AWS documentation.
2. We applied the Terraform to create the new Managed Node Groups, without removing the old Auto Scaling Groups
3. When the new nodes had finishing coming online we drained the old ones with this handy one-liner:
  ```bash
  kubectl drain --selector '!eks.amazonaws.com/nodegroup' --delete-local-data --ignore-daemonsets
  ```
4. We then removed the old Auto Scaling Groups from the Terraform and applied it to delete the old, now empty nodes.

I hope this post helps those of you who haven't yet moved to Managed Node Groups. If they don't yet meet your requirements I'd love to hear what weird and wonderful things you're doing. I'm always happy to chat to people working on similar challenges, so get in touch on [Twitter](https://twitter.com/cablespaghetti) if you have any questions or just fancy a chat!
