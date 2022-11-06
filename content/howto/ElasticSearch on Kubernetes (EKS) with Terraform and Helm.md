---
title: (WIP) Creating a Kubernetes cluster on EKS with Terraform
author: Marius Kimmina
date: 2022-10-30
tags: [Kubernetes, AWS, EKS]
published: true
---

As someone who uses K8s (and AWS) all the time at work, I often find myself wanting to try out some things on my own before I bring those ideas up at work. 
I therefore need access to a cluster of my own, which should of course be much smaller and easy to throw away once I don't need it anymore as I don't want to break the bank for my experiements.

In this article I will show you how I created and manage a small Kubernetes cluster and on EKS with Terraform that I can quickly spin up and down. Future articles will showcase the actual experiments that I run on this cluster.

I assume that anyone who reads this has at least some basic understanding of Kubernetes and AWS, I will therefore not waste time explaining you what these things are or how to create an AWS account for example, I trust you can figure that out.

**Requirements:**
* AWS account
* aws-cli setup
* terraform installed


## Terraform repo

We start with setting up a repository for our Terraform code right away. One of the most important aspects of Infrastructure as Code is to be consistent with it and **never give in to the urge of doing small things manually**. Everything we want to do on AWS has to happen through Terraform.

```bash
mkdir my-little-eks
cd my-little-eks
```

Blabla Version control

```bash
git init 
```

Blabla basic Terrarom structure

```bash
$ tree
.
├── cluster.tf
├── main.tf
├── outputs.tf
├── sec-groups.tf
├── variables.tf
├── versions.tf
└── vpc.tf

0 directories, 7 files
```

Let's first look at the `versions.tf` file. Here I define which verions of the `terraform` CLI and aws-provider are required to use this project.

```tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.15.0"
    }
  }
  required_version = ">= 1.2.0"
}
```

The actual verions of the `terraform` CLI that I am currently on is `1.3.4` but I expect this to work with anything newer than `1.2.0`. 

Next up is the `variables.tf` file. This one is also kept short, currently I am only defining the AWS region and the name for the cluster in here.

```tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "eu-central-1"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "my-little-eks"
}
```

---
main

```
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
}

provider "aws" {
  region = var.region
}

data "aws_availability_zones" "available" {}

locals {
  cluster_name = "education-eks-${random_string.suffix.result}"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
}
```

---
vpc

```
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.2"

  name = "my-little-vpc"

  cidr = "10.0.0.0/16"
  azs  = slice(data.aws_availability_zones.available.names, 0, 3)

  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = 1
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = 1
  }
}
```

---
cluster

```
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "18.26.6"

  cluster_name    = local.cluster_name
  cluster_version = "1.22"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_group_defaults = {
    ami_type = "AL2_x86_64"

    attach_cluster_primary_security_group = true

    # Disabling and using externally provided security groups
    create_security_group = false
  }

  eks_managed_node_groups = {
    one = {
      name = "node-group-1"

      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 3
      desired_size = 2

      pre_bootstrap_user_data = <<-EOT
      echo 'foo bar'
      EOT

      vpc_security_group_ids = [
        aws_security_group.node_group_one.id
      ]
    }

    two = {
      name = "node-group-2"

      instance_types = ["t3.medium"]

      min_size     = 1
      max_size     = 2
      desired_size = 1

      pre_bootstrap_user_data = <<-EOT
      echo 'foo bar'
      EOT

      vpc_security_group_ids = [
        aws_security_group.node_group_two.id
      ]
    }
  }
}
```

---
outputs

```
output "cluster_id" {
  description = "EKS cluster ID"
  value       = module.eks.cluster_id
}

output "cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = module.eks.cluster_endpoint
}

output "cluster_security_group_id" {
  description = "Security group ids attached to the cluster control plane"
  value       = module.eks.cluster_security_group_id
}

output "region" {
  description = "AWS region"
  value       = var.region
}

output "cluster_name" {
  description = "Kubernetes Cluster Name"
  value       = local.cluster_name
}
```



