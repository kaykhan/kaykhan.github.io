---
layout: post
title:  "How to maintain high availibity when using spot instances on AWS EKS (Solving un-balanced pods)"
date:   2023-06-06 13:00:00 +0100
categories: aws eks spot spot-instances 
---

Spot instances are instances that use spare EC2 capacity and becuase of that they are much cheaper than on-demand pricing.
[Learn more](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html). However they can be reclaimed by AWS within a 10 minute window when someone purchases an on-demand instance.

We have an AWS EKS cluster, there is a particular workload which is suitable to be deployed on spot instances nodes. 

## Goal

How do we define a set of AWS EKS nodes to be of type spot-instances as appose to on-demand instances. And how do we assign specific workload to be deployed on these spot instance nodes.


## Architecture



## Code


