---
published: true
date: '2025-01-28 14:02 +0530'
title: Deploying cnBNG Control Plane in Redhat Openshift
author: Gurpreet Dhaliwal
position: hidden
---

{% include toc %}

## Overview

cnBNG Control Plane can be deployed in Redhat Openshift environment starting from release 25.01.1. Current supported Openshift Container Platform (OCP) version is v4.17.  In this tutorial we will learn to deploy cnBNG Control Plane in Single Node Openshift cluster. 

Deployment of cnBNG Control Plane cluster in Openshift environment has three steps:

1. Preparing Openshift Container Platform (OCP)
1. CEE Deployment for Monitoring and essential Cloud Native Applications
1. cnBNG Control Plane Deployment

## Prerequisite

Following are the prerequisite for this tutorial:

- Openshift Container Platform (OCP) deployed in either a VM or on a Baremetal Server
- Helm Chart Repository hosting Control Plane helm charts
- Docker Image Repository hosting docker images for Control Plane

Note: We will cover Helm Chart Repository and Docker Image Repository creation in some other tutorial.

## Network

In this tutorial we will use single network for management as well as to peer with User Planes and communicate with Radius. You can also use separate networks for management,  User Plane peering and Radius communication.

## Preparing OCP Environment

1. Login to local OCP cluster console as admin.
1. Now, add following labels to the node. These labels are required for cnBNG POD deployment/ placement. Add labels by selecting Nodes under Compute from side panel.
  ```
  smi.cisco.com/node-type=oam
  disktype=ssd
  ```
  ![Screenshot 2025-01-28 at 3.57.02 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-28 at 3.57.02 PM.png)
  ![Screenshot 2025-01-28 at 3.57.06 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-28 at 3.57.06 PM.png)
1. Create two namespaces "ceeocp" and "bngocp" by selecting Namespaces under Administration from side panel
	![Screenshot 2025-01-29 at 12.02.45 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.02.45 PM.png)
	![Screenshot 2025-01-29 at 12.03.17 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.03.17 PM.png) | ![Screenshot 2025-01-29 at 12.03.45 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.03.45 PM.png)
1. Switch to the Developer mode from Administrator mode by selecting it from Side Panel
	![Screenshot 2025-01-29 at 12.14.27 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.14.27 PM.png)
1. 

