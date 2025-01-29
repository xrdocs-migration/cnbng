---
published: true
date: '2025-01-28 14:02 +0530'
title: Deploying cnBNG Control Plane in Redhat Openshift
author: gurpreets
position: hidden
---
## Overview

cnBNG Control Plane can be deployed in Redhat Openshift environment starting from release 25.01.1. Current supported Openshift Container Platform (OCP) version is v4.17.  In this tutorial we will learn to deploy cnBNG Control Plane in Single Node Openshift cluster. 

Deployment of cnBNG Control Plane cluster in Openshift environment has three steps:
	1. Preparing Openshift Container Platform (OCP)
	2. CEE Deployment for Monitoring and essential Cloud Native Applications
	3. cnBNG Control Plane Deployment

## Prerequisite

	- Openshift Container Platform (OCP) deployed in either a VM or on a Baremetal Server
	- Helm Chart Repository 
	- Docker Image Repository

Note: We will cover Helm Chart Repository and Docker Image Repository creation in some other tutorial.

## Network

In this tutorial we will use single network for management as well as to peer with User Planes and communicate with Radius. You can also use separate networks for management,  User Plane peering and Radius communication.

## Preparing OCP Environment

1. As a first step we will add labels to the node. These labels are required for cnBNG POD deployment/ placement. Add following labels to the node by clicking on Edit Compute->Nodes (see below)

```
smi.cisco.com/node-type=oam
disktype=ssd
```

![Screenshot 2025-01-28 at 3.57.02 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-28 at 3.57.02 PM.png)

![Screenshot 2025-01-28 at 3.57.06 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-28 at 3.57.06 PM.png)
