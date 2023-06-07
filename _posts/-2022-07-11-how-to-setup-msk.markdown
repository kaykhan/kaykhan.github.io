---
layout: post
title:  "(WIP) How to setup & connect to AWS MSK Kafka"
date:   2022-07-11 16:21:50 +0100
categories: aws msk kafka 
---

Amazon Managed Streaming for Apache Kafka (MSK) Securely stream data with a fully managed, highly available Apache Kafka service
[Learn more](https://aws.amazon.com/msk/).

## 1. Setup

We will use terraform to create an AWS MSK cluster. Assumes you know the basics of terraform and already have a VPC.

### Features
1. Logging eanbled and ship to cloudwatch
2. SG created to allow access from within VPC
3. Create an S3 bucket for plugins 

msk/main.tf
```
locals {
  workspace = terraform.workspace
  name      = "${var.org_name}-${local.workspace}-msk"
  region    = var.aws_region
  tags = {
    Owner       = "acme"
    Environment = local.workspace
    Terraform   = "true"
  }
}

resource "aws_msk_cluster" "msk_cluster" {
  cluster_name           = local.name
  kafka_version          = "3.2.0"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type  = "kafka.t3.small"
    client_subnets = data.terraform_remote_state.networking.outputs.public_subnets
    storage_info {
      ebs_storage_info {
        volume_size = 100
      }
    }
    security_groups = [aws_security_group.msk_sg.id]
  }

  client_authentication {
    sasl {
      iam = true
    }
  }

  open_monitoring {
    prometheus {
      jmx_exporter {
        enabled_in_broker = true
      }
      node_exporter {
        enabled_in_broker = true
      }
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk_cluster.name
      }
    }
  }

  depends_on = [
    aws_security_group.msk_sg,
    aws_cloudwatch_log_group.msk_cluster
  ]


  tags = local.tags
}


resource "aws_cloudwatch_log_group" "msk_cluster" {
  name = "${local.name}-logs"
  tags = local.tags
}

resource "aws_s3_bucket" "msk_plugins" {
  bucket = "acme-msk-plugins"

  tags = {
    Name = "ACME MSK Plugins"
  }
}

resource "aws_s3_bucket_acl" "msk_plugins" {
  bucket = aws_s3_bucket.msk_plugins.id
  acl    = "private"
}

resource "aws_security_group" "msk_sg" {
  name   = "msk-sg-${local.workspace}"
  vpc_id = data.terraform_remote_state.networking.outputs.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 9098
    to_port     = 9098
    protocol    = "tcp"
    description = "MSK access from within VPC"
    cidr_blocks = [data.terraform_remote_state.networking.outputs.vpc_cidr_block]
  }

  lifecycle {
    ignore_changes = [
      ingress
    ]
  }

  tags = {
    Name      = "${var.org_name}-${terraform.workspace}-msk-sg"
    Terraform = true
  }

}

```

## 2. Connect

1. Make sure your aws user has the right permissions 
2. create client.properties
3. Download jar and set classpath
