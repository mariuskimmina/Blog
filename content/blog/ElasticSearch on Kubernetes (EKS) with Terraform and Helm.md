---
title: Creating a Kubernetes cluster on EKS with Terraform
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

We start with setting up a repository for our Terraform code right away. One of the most important aspects of Infrastructure as Code is to be consistent with it and **never** give in to the urge of doing small things manually. Everything we want to do on AWS has to happen through Terraform.

```
mkdir my-little-eks
cd my-little-eks
```

Blabla Version control

```
git init 
```

Blabla basic Terrarom structure

```
touch main.tf
touch versions.tf
touch variables.tf
touch output.tf
```

Let's first look at the verisons.tf. Here I define the expected version of Terraform and the aws-provider. 

```
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

The actual verion of `terraform` that I am currently on is 1.3.4

