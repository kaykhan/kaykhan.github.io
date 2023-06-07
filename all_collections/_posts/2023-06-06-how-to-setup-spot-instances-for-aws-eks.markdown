---
layout: post
title:  "How to setup spot instances on AWS EKS"
date:   2023-06-06 11:00:00 +0100
categories: aws eks spot terraform
---

We use AWS Kubernetes offering "EKS". There is a particualar workload which is suitable usecase for spot instances

## Goal

How do we define a set of AWS EKS nodes to be of type spot-instance as appose to on-demand instance. And how do we assign specific workload to be deployed on these spot instance nodes.

## Architecture

We have a kubernetes cluster which consits of 3 different node groups. `worker`, `cron` and `ops`. We determined that the `worker` node group which consists of two nodes is sutiable for spot instances.

![architecture](/images/spot-instance/spot-instance-architecture-on-aws-eks.png)


## Code

<b>1. Setup Spot Instance Node Group Using Terraform</b>

We use the terraform [AWS EKS](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest) module to deploy a production kubernetes cluster to AWS.

- We define a set of `eks_managed_node_groups`. Given our architecture we need 3 node groups (worker, cron and ops). Where the `worker` node group will be a spot instance node group. 

- We use the `capacity_type` parameter to set whether the node group will be of type `"SPOT"` or `"ON_DEMAND"`

- We can set multiple `instance_types` from differnet ec2 families all with the same underlining hardware resource (CPU & Memory) to ensure that we are always allocated an ec2 instance. 
We can use the cli tool [amazon-ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector) to help us find ec2 instance types that meet our requirement.

- We set a custom label on each node group called `acme/node-type` which we can then later reference when deploying our kubernetes resources

{% highlight terraform %}

module "eks" {
  source                          = "terraform-aws-modules/eks/aws"
  version                         = "19.13.0"
  cluster_name                    = local.cluster_name
  cluster_version                 = local.cluster_version
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

 ...

  eks_managed_node_groups = {

    "${local.cluster_name}-worker-spot" = {
      iam_role_use_name_prefix = false
      ami_type                 = "AL2_x86_64"
      min_size                 = 2
      max_size                 = 10
      desired_size             = 2
      instance_types           = ["m5.xlarge", "m4.xlarge", "m5a.xlarge", "m5ad.xlarge", "m5d.xlarge", "m6i.xlarge", "t3.xlarge", "t3a.xlarge"]
      capacity_type            = "SPOT"
      ebs_optimized            = true
      labels = {
        "acme/node-type" = "worker"
        Environment       = local.workspace
      }

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 50
            volume_type           = "gp3"
            delete_on_termination = true
          }
        }
      }
      tags = local.tags
    }

    "${local.cluster_name}-ops" = {
      iam_role_use_name_prefix = false
      ami_type                 = "AL2_x86_64"
      min_size                 = 3
      max_size                 = 10
      desired_size             = 3
      instance_types           = ["m5.2xlarge"]
      capacity_type            = "ON_DEMAND"
      ebs_optimized            = true
      subnet_ids               = data.aws_subnets.ops_group_subnets.ids
      labels = {
        "acme/node-type" = "ops"
        Environment       = local.workspace
      }

      tags = local.tags
    },

    "${local.cluster_name}-cron" = {
      iam_role_use_name_prefix = false
      ami_type                 = "AL2_x86_64"
      min_size                 = 1
      max_size                 = 10
      desired_size             = 1
      instance_types           = ["m5.xlarge"]
      capacity_type            = "ON_DEMAND"
      ebs_optimized            = true
      labels = {
        "acme/node-type" = "cron"
        Environment       = local.workspace
      }

      tags = local.tags
    },
  }

}

{% endhighlight %}


<b>2. Assign Kubernetes workload to be deployed on the spot instance</b>

When creating a kubernetes deloymenet we can use the pod spec `nodeSelector` to allocate/schedule this deployment to a specific nodegroup. [learn more](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

{% highlight terraform %}
spec:
    nodeSelector:
        acme/node-type: worker
{% endhighlight %}


## References

- <a href="https://aws.amazon.com/ec2/spot/">https://aws.amazon.com/ec2/spot/</a>
- <a href="https://repost.aws/knowledge-center/eks-spot-instance-best-practices">https://repost.aws/knowledge-center/eks-spot-instance-best-practices</a>
- <a href="https://aws.amazon.com/blogs/compute/run-your-kubernetes-workloads-on-amazon-ec2-spot-instances-with-amazon-eks/">https://aws.amazon.com/blogs/compute/run-your-kubernetes-workloads-on-amazon-ec2-spot-instances-with-amazon-eks/</a>
- <a href="https://aws.amazon.com/blogs/compute/cost-optimization-and-resilience-eks-with-spot-instances/">https://aws.amazon.com/blogs/compute/cost-optimization-and-resilience-eks-with-spot-instances/</a>
- <a href="https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html">https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html</a>
