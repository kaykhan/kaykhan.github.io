---
layout: post
title:  "(WIP) How to solve unbalanced pods when spot instances are reclaimed on AWS EKS using Descheduler"
date:   2023-06-06 13:00:00 +0100
categories: aws eks spot scheduler balance pods descheduler terraform
---

We have recently swapped one of our node groups which consists of 2 nodes from On-Demand instance type to Spot instance type. 

We have noticed some unexpected behaviour after a spot instance is reclaimed. We notice that on a 2 node setup with a deployment of 3 replicas. The pods are likely to consolidate to a single node. 

When that node eventually is relciamed we start to notice downtime of our services. We end up losing High Availability.

We looked at the current scheduling features kubernetes provides; `affinity`, `podAntiAffinity` and `topologySpreadConstraint` but on their own did not solve our problem.

## Goal

We would like to maintain high availiblity of our services.
How do we enforce balacing of pods throughout the life cycle of a deployment, including after a spot instance is reclaimed.

## Solution 

We need something that will affect <b>post-scheduling</b> behaviour of pods this is when we found - [descheduler](https://github.com/kubernetes-sigs/descheduler). Descheduler has a number of policies which effect the behaviour of kubernetes resource post-scheduling by contiously monitoring the kubernetes resources. 


## Code


