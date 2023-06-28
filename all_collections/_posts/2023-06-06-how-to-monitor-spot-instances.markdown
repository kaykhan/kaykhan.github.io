---
layout: post
title:  "How to monitor spot instance changes on AWS"
date:   2023-06-06 12:00:00 +0100
categories: aws eks spot event-bridge sns lambda slack terraform
---

Spot instances are instances that use spare EC2 capacity and becuase of that they are much cheaper than on-demand pricing. However they can be reclaimed by AWS within a 10 minute when there are requests/purchases for on-demand instance [(learn more)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html).

We have an AWS EKS cluster, there is a particular workload which is suitable to be deployed on spot instances. Recently we swapped this workload from on-demand instances to spot instances. 

## Goal

We need some way to monitor when the spot instances are reclaimed.

How do we define spot instance notifications & can we ship these notifications to an aws slack channel? 


## Architecture

![architecture](/images/spot-instance/spot-instance-monitoring-architecture.png)

<b>1. Create Event Bridge Rules for Spot Instance Changes</b>

Event Bridge is a aws severless component which allows us to build event-driven architecture.

Within Event Bridge there are number of Event Types dedicated to Spot Instances.
[Learn More](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/spot-fleet-event-types.html)

- EC2 Spot Fleet State Change
- EC2 Spot Fleet Spot Instance Request Change
- EC2 Spot Fleet Instance Change
- EC2 Spot Fleet Information
- EC2 Spot Fleet Error


<b>2. When an event Bridge Rule is triggered send the Event to an SNS topic</b>

<b>3. A Lambda functions listens to the SNS Topic for any new messages</b>

<b>4. The Lambda function publishes each message to a Slack Channel</b>

## Code

<b>1. Setup SNS Topic & Lambda</b>

We can use the terraform module [Notify Slack](https://registry.terraform.io/modules/terraform-aws-modules/notify-slack/aws/latest) to setup the SNS topic & Lambda function. This module does a lot of the heavy lifting for us, no need to write any custom lambda code.
We can use Notify Slack to publish different messages but in our case we will be using it for Event Bridge Rules.

{% highlight terraform %}

module "notify_slack" {
  source  = "terraform-aws-modules/notify-slack/aws"
  version = "~> 5.0"

  sns_topic_name = "slack-topic"

  slack_webhook_url = local.slack_webhook_url 
  slack_channel     = "aws"
  slack_username    = "AWS"

  lambda_attach_dead_letter_policy = true
  lambda_dead_letter_target_arn = aws_sqs_queue.notify_slack_dlq.arn	
}

{% endhighlight %}

<b>2. Define Event Bridge Rules</b>

Using the terraform aws module [EventBridge](https://registry.terraform.io/modules/terraform-aws-modules/eventbridge/aws/latest).
We can define a rule for each spot instance event type which we would like to track. Aswell as defining a target whenever the rule is triggered it sends the message to the target (SNS topic).

{% highlight terraform %}

resource "aws_sqs_queue" "event_bridge_dlq" {
  name = "event_bridge_dlq"
  tags = local.tags
}


module "eventbridge" {
  source  = "terraform-aws-modules/eventbridge/aws"
  version = "2.3.0"

  create_bus         = false
  create_permissions = true
  create_role        = true

  attach_sns_policy = true
  sns_target_arns = [
    local.slack_topic_arn
  ]

  rules = {
    ec2-spot-instance-request-fullfillment = {
      description = "ec2-spot-instance-request-fullfillment"
      event_pattern = jsonencode({
        "source" : ["aws.ec2"],
        "detail-type" : ["EC2 Spot Instance Request Fulfillment"]
      })
      enabled = true
    }

    ec2-spot-interruption-warning = {
      description = "ec2-spot-interruption-warning"
      event_pattern = jsonencode({
        "source" : ["aws.ec2"],
        "detail-type" : ["EC2 Spot Instance Interruption Warning"]
      })
      enabled = true
    }

    ec2-spot-fleet-instance-change = {
      description = "ec2-spot-fleet-instance-change"
      event_pattern = jsonencode({
        "source" : ["aws.ec2fleet"],
        "detail-type" : ["EC2 Fleet Spot Instance Request Change"]
      })
      enabled = true
    }

    spot-fleet-instance-change = {
      description = "spot-fleet-instance-change"
      event_pattern = jsonencode({
        "source" : ["aws.ec2spotfleet"],
        "detail-type" : ["EC2 Spot Fleet Instance Change"]
      })
      enabled = true
    }

    spot-fleet-instance-request-change = {
      description = "spot-fleet-instance-request-change"
      event_pattern = jsonencode({
        "source" : ["aws.ec2spotfleet"],
        "detail-type" : ["EC2 Spot Fleet Spot Instance Request Change"]
      })
      enabled = true
    }

    spot-fleet-state-change = {
      description = "spot-fleet-state-change"
      event_pattern = jsonencode({
        "source" : ["aws.ec2spotfleet"],
        "detail-type" : ["EC2 Spot Fleet State Change"]
      })
      enabled = true
    }

    spot-fleet-information = {
      description = "spot-fleet-information"
      event_pattern = jsonencode({
        "source" : ["aws.ec2spotfleet"],
        "detail-type" : ["EC2 Spot Fleet Information"]
      })
      enabled = true
    }

    spot-fleet-error = {
      description = "spot-fleet-error"
      event_pattern = jsonencode({
        "source" : ["aws.ec2spotfleet"],
        "detail-type" : ["EC2 Spot Fleet Error"]
      })
      enabled = true
    }

  }

  targets = {

    ec2-spot-instance-request-fullfillment = [
      {
        name            = "ec2-spot-instance-request-fullfillment"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      }
    ]
    ec2-spot-interruption-warning = [
      {
        name            = "ec2-spot-interruption-warning"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      }
    ]

    ec2-spot-fleet-instance-change = [
      {
        name            = "ec2-spot-fleet-instance-change"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      }
    ]

    spot-fleet-instance-change = [
      {
        name            = "spot-fleet-instance-change"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      },
    ],
    spot-fleet-instance-request-change = [
      {
        name            = "spot-fleet-instance-request-change"
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
        arn             = local.slack_topic_arn
      },
    ],
    spot-fleet-state-change = [
      {
        name            = "spot-fleet-state-change"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      },
    ],
    spot-fleet-information = [
      {
        name            = "spot-fleet-information"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      },
    ],
    spot-fleet-error = [
      {
        name            = "spot-fleet-error"
        arn             = local.slack_topic_arn
        dead_letter_arn = aws_sqs_queue.event_bridge_dlq.arn
      },
    ]
  }

  tags = local.tags
}

{% endhighlight %}

That it! When the rules are triggered they will be published to the SNS topic, the lambda function will listen to new messages on that topic and publish them to slack.
