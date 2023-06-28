---
layout: post
title:  "Reduce the cost of ec2 compute using spot instances"
date:   2023-06-06 10:00:00 +0100
categories: spot ec2 startup
---

We have been running EC2 m5.xlarge instances for 2 years and have always used the on-demand instance type. On-demand instances are uninterrupted but we pay for this with a static hourly price. 

Spot instances are instances that use spare EC2 capacity meaning they are much cheaper than on-demand instances and their pricing is dynamic. However they can be reclaimed by AWS within a 10 minute window when there are requests/purchases for on-demand instance [(learn more)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html).

Using Spot instances with their dyanmic pricing we can receive a discount of up to 90% depending on availability of spare ec2 capacity. The potential discount varies daily/weekly/monthly and depends on the base instance type.

We identified a workload on our AWS EKS which is sutiable for spot instances. We have a node group `worker` where we deploy our APIs. These APIs are suitable match for spot instances as they are flexible and fualt tolerent.


## Cost Savings Estimate

We can use [AWS's Pricing Calculator](https://calculator.aws/#/) to estimate the potential cost savings.

Given a node group with 2 nodes which run m5.xlarge and the average discount of 75%, we are estiamted to save.

- 2x m5.xlarge OnDemand: $308.52 / month
- 2x m5.xlarge Spot: $76.28 / month

- Monthly Savings: $308.52 - $76.28 = <b>$232.24</b>
- Yearly Savings: $3702.24 - $915.36 = <b>$2786.88</b>

<small>These numbers are estimates and likely to change</small>


## References

- <a href="https://docs.aws.amazon.com/whitepapers/latest/cost-optimization-leveraging-ec2-spot-instances/when-to-use-spot-instances.html">https://docs.aws.amazon.com/whitepapers/latest/cost-optimization-leveraging-ec2-spot-instances/when-to-use-spot-instances.html</a>
- <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html">https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html</a>
