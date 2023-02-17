---
title: WIP Building better CI/CD Pipelines
author: Marius Kimmina
date: 2022-09-02 14:10:00 +0800
tags: [Infrastructure, Go, Cloudflare]
published: false
---

If you are reading this, you probably know what a CI/CD Pipeline is and you have probably used one before. You 
Short disclaimer: I have only been using Gitlab pipelines for all of my career and I will be using Gitlab for all examples in this Post. That being said, most priciples you find here can also be implemented in other CI/CD platforms like TravisCI or Github Actions.


## When to run pipelines

When a repo consists of more than 1 application, a so called [monorepo](https://example.com), then we want to run tests on the application that has changes. Always testing all applications in a repo with potentially 10s or even 100s of them can be unfathomably time consuming. If you are using cloud service to run your Pipeline you might also have to pay more for the heavier load.

Many CI/CD platforms offer a way to check for changes to certain files or directorys before running a pipeline. In Gitlab you can do it as follows:

```yaml
only:
  changes:
    - dir-a/*
    - dir-b/**/*
    - file-c
```

This defines a pipeline that is only executed when a) a file in `dir-a` has been changed, b) a file in `dir-b` or any subdirectory of `dir-b` has been changed, c) `file-c` has been changed.

Be aware that `changes` in Gitlab is always considered true when the pipeline is triggered manually instead of by pushing code. 


Another case I come accross frequently is to manually trigger only certain parts of the pipeline and all CI/CD platfoms shoud allow for this in one way or another. In Gitlab you can define custom variables before triggering a Pipeline manually and then check these variables for running certain jobs.

For example, if I have two seperate test jobs defined, `Test A` und `Test B` then I can define that `Test A` may only run if the variable `TEST_A` is set to `yes`

```yaml
only:
  variables: 
    - $TEST_A == "yes"
```

When you define many rules around when to run your pipelines, things can get a bit confusing, for example consider the following `only` definition. This Pipeline will run when either `TEST_A` or `TEST_C` is yes. 

```yaml
only:
  variables: 
    - $TEST_A == "yes"
    - $TEST_C == "yes"
```

But what about the following? 

```yaml
only:
  variables: 
    - $TEST_A == "yes"
    - $TEST_C == "yes"
  changes:
    - dir-a/*
    - dir-b/**/*
    - file-c
```

Well this will also run when the Pipeline is triggerd manually and either `TEST_A` or `TEST_C` is set to yes. It will never on any code push tho, because both the `variables` and the `changes` section have to evaluate to `true`. When you trigger the Pipeline manually, `changes` are always true, so in that case it works, but if you push code changes you never have custom variables defined thus your pipeline will never run.


---


  # this is an AND
  # It always works with variables because changes for manually started pipelines always returns true ...
      #only:
      #variables: 
      #- $TEST_DSP == "yes"
      #- $TEST_ALL == "yes"
      #changes:
      #- dsp-dashboard/**/*
      #- dashboard-shared/**/*

What worked:
  rules:
    - if: '$TEST_DSP == "yes" || $TEST_ALL == "yes"'
    - if: '$TEST_SSP == "yes" || $TEST_BETA == "yes"'
      when: never
    - changes:
        paths:
        - dsp-dashboard/**/*
        - dashboard-shared/**/*

This is a bit ugly tho.
