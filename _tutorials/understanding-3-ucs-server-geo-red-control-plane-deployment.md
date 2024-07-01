---
published: true
date: '2024-07-01 14:30 +0530'
title: Understanding 3 UCS Server Geo-Red Control Plane Deployment
author: Gurpreet Singh
excerpt: >-
  This tutorial explains the concepts related to three server Geo Redundant
  Control Plane Baremetal/CNDP cluster Deployment. 
tags:
  - cisco
  - cnbng
  - deployment
---
## Overview

cnBNG Control Plane can be categorised based on Kubernetes cluster as either single node cluster deployment or as multi node cluster deployment. The node here can refer to as:
	- VM, or
	- A Baremetal Server (often UCS)

Deploying Control Plane in Baremetal form is beneficial and can save upto 30% of CPU because resources are not required to be reserved for Hypervisor. Another advantage of running Control Plane on baremetal servers is that SMI (Cisco Ultra Cloud Core) can monitor server performance along with Control Plane performance. 

In this tutorial we will learn about Geo Redundant Control Plane multi node cluster deployment. Multi-node cluster has unique requirements from networking perspective. And Geo redundancy adds additional networking overhead which could be complex to understand at first. We will try to demystify this and look at how we can leverage Subscriber Edge deployment tool (https://github.com/xrdocs/subscriber-edge) to ease the deployment process. 

## Layered Control Plane Architecture

We all know that Cisco Cloud Native BNG Control Plane is designed as a layered architecture for better resiliency and scalability. Fig.1. below shows the layered control plane architecture. The layered architecture works on Kuberenetes node labelling mechanism. Which allows to spawn containers/ PODs to respective servers/nodes based on the labels. Efficient and careful labelling can provide protection against POD, server and link failure.

We use following K8s labels in a typical multi-node cluster deployment of control plane. These can be categorised into four layers: OAM, Protocol, Service and Session. 

```
k8s node-labels smi.cisco.com/node-type oam
k8s node-labels smi.cisco.com/proto-type protocol
k8s node-labels smi.cisco.com/protocol protocol
k8s node-labels smi.cisco.com/svc-type service
k8s node-labels smi.cisco.com/sess-type cdl-node
```
