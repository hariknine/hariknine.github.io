---
title: GPU Partitioning on GKE
published: 2025-11-20
description: Guide to Partition GPU nodes on GKE - MIG and Time-Slicing
tags: [GKE, Kubernetes, GPU]
category: Docs
draft: true
prevTitle: "GitHub actions with AWS IAM Role"
prevSlug: "gh-actions-aws-role"
---
GPU sharing in Kubernetes denotes paritioning GPU nodes (software or hardware level) so that containers can packed together and use the GPU nodes more efficiently. For example - a GPU workload can be deployed in an `A100` node with the resource request / limit - `nvidia.com/gpu: "1"`. This means the workload is occupying the whole GPU even though it probably isn't using all of it. Unlike CPU and Memory, GPU units can't be fractionally divided. Hence we must use various GPU sharing methods like :
* [MIG sharing](#mig)
* [Time-Slicing](#time-slicing)
* [MPS](#mps)

# MIG
This is a hardware-level partitioning method, primarily available in data center GPUs - 
* `AWS` - A100, H100, B200, GB200 - [Ref](https://docs.aws.amazon.com/dlami/latest/devguide/gpu.html)
* `GKE` - A100, H100, H200 B200, GB200, RTX PRO 6000 - [Ref](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/gpus-multi#supported_gpus)
# Time-Slicing

# MPS