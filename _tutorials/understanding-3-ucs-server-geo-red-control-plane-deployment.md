---
published: true
date: '2024-07-01 14:30 +0530'
title: Understanding 3 UCS Server Geo-Red Control Plane Deployment
author: Gurpreet Dhaliwal
excerpt: >-
  This tutorial explains the concepts related to three server Geo Redundant
  Control Plane Baremetal/CNDP cluster Deployment. 
tags:
  - cisco
  - cnbng
  - deployment
position: hidden
---
{% include toc %}

## Overview

cnBNG Control Plane cluster deployment can be categorised as a single node cluster deployment, or as a multi node cluster deployment. The node here can refer to as:
- VM, or
- A Baremetal Server (often UCS)

Deploying Control Plane in Baremetal form is beneficial and can save upto 30% of CPU, because resources are not required to be reserved for Hypervisor. Another advantage of running Control Plane on baremetal server(s) is that SMI (Cisco Ultra Cloud Core) can monitor server performance along with Control Plane performance. 

In this tutorial we will learn concepts about Geo Redundant Control Plane multi-node cluster deployment. Multi-node cluster has unique requirements from networking perspective, and Geo redundancy adds additional networking overhead which could be complex to understand at first. We will try to demystify this.

With the help of Subscriber Edge deployment tool (https://github.com/xrdocs/subscriber-edge) it is easy to generate configuration for SMI Cluster Deployer and deploy the Multi Node cluster easily.

## Layered Control Plane Architecture

We all know that Cisco Cloud Native BNG Control Plane is designed as a layered architecture for better resiliency and scalability. Fig.1. below shows the layered control plane architecture. The layered architecture works on Kuberenetes node labelling mechanism. Which allows to spawn containers/ PODs to respective servers/nodes based on the labels. Efficient and careful labelling can provide protection against POD, server and link failure.

![cp_architecture.png]({{site.baseurl}}/images/cp_architecture.png){: .align-center}{:height="60%" width="60%"}
Fig.1. Layered Control Plane Architecture
{: .text-center}


We use following K8s labels in a typical multi-node cluster deployment of control plane. These can be categorised into four layers: OAM, Protocol, Service and Session. 

```
k8s node-labels smi.cisco.com/node-type oam
k8s node-labels smi.cisco.com/proto-type protocol
k8s node-labels smi.cisco.com/protocol protocol
k8s node-labels smi.cisco.com/svc-type service
k8s node-labels smi.cisco.com/sess-type cdl-node
```

## Three Server Geo Redundant Control Plane Cluster Logical Topology

In Fig.2. below, you can see different networks and logical connectivity between servers. Also notice that how servers are categorized into four layers of Control Plane architecture.

![3svr_logical_topo.png]({{site.baseurl}}/images/3svr_logical_topo.png){: .align-center}
Fig.2. Geo Redundant Control Plane Cluster Logical Topology
{: .text-center}

In Fig.2., service network svc-net1 (or n4) is created for communication with User Planes and AAA. cnBNG Control Plane can have a separate service network to communicate with AAA. 

Three networks inttcp, intudp and cdl are created for Geo Redundancy. In stand-alone cluster deployment, these networks are not required. 

Fig.3. below shows interfaces (cdl and inttcp) used for synchronization between two clusters. As many nodes are involved in the cluster and different IPs are assigned including Virtual IPs, entire subnet (/28) needs to be accessible from remote end for these two interfaces. 

![geored_connections.png]({{site.baseurl}}/images/geored_connections.png){: .align-center}{:height="60%" width="60%"}
Fig.3. Interfaces for inter cluster synchronization
{: .text-center}

## Three Server Geo Redundant Control Plane Cluster Phyical Topology

Physical topology of three server baremetal cluster is shown in Fig.4. below. Every cluster can have different physical mapping based on datacenter design restrictions. Below are Cisco recommended cluster topology connections for greater resiliency.

![3svr_physical_topo.png]({{site.baseurl}}/images/3svr_physical_topo.png){: .align-center}
Fig.4. Geo Redundant Control Plane Cluster Phyical Topology
{: .text-center}

Cisco recommends creating bond interfaces to provide redundancy for link failures, especially for k8s and management networks. Here are the bond interfaces which are created in a typical three server deployment model:

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>Bond Interface</th>
    <th>Links</th>
  </tr>
  <tr>
    <td>bd0</td>
    <td>eno5 and eno6 (MLOM Ports)</td>
  </tr>
  <tr>
    <td>bd1</td>
    <td>enp94s0f1 (PCIE1 port)</td>
  </tr>
  <tr>
    <td>bd2</td>
    <td>enp94s0f0 (PCIE2 port</td>
  </tr>
</table>

Apart from these two interfaces are used to form eBGP sessions. These are PCIE1 and PCIE2 interfaces from UCS01 and UCS02 protocol nodes. In a typical cluster deployment, it is recommended to form two EBGP sessions for redundancy. 

<table style="width:100%" border = "2">
  <tr bgcolor="lightblue">
    <th>EBGP Session</th>
    <th>Link</th>
  </tr>
  <tr>
    <td>ebgp1</td>
    <td>enp216s0f0</td>
  </tr>
  <tr>
    <td>ebgp2</td>
    <td>enp94s0f0</td>
  </tr>
</table>

Here is the logical interface to physical port mapping and an example VLAN map:

```
    networks:
      k8s:
        id: 125  # VLAN ID
        intf: bd0
      mgmt:
        id: 325  # VLAN ID
        intf: bd0
      inttcp:
        id: 164  # VLAN ID
        intf: bd1
      intudp:
        id: 163  # VLAN ID
        intf: bd2
      n4:
        id: 161  # VLAN ID
        intf: bd2
      cdl:
        id: 165  # VLAN ID
        intf: bd1
```

## Virtual IP Addresses

In a three server cluster deployment, virtual IP addresses (created for VRRP between nodes) helps POD/containers reach respective services. Here is the list of virtual IPs which are created for cluster operations under respective network interfaces.

```
    networks:
      inttcp:
        id: 164  # VLAN ID
        intf: bd1
        vip1: 1.1.164.100
        vip2: 1.1.164.101
        broadcast: 1.1.164.255
      intudp:
        id: 163  # VLAN ID
        intf: bd2
        vip1: 1.1.163.100
      n4:
        id: 161  # VLAN ID
        intf: bd2
        vip1: 72.72.100.1  # this IP (advertised by BGP) is used to connect UPs with Instance-1 of this cluster
        vip2: 73.73.100.2  # this IP (advertised by BGP) is used to connect UPs with Instance-2 of this cluster
      cdl:
        id: 165  # VLAN ID
        intf: bd1
```

## Cluster IP Addressing Map/ Table

Below is an IP addressing Map or Table to help decide on IP addresses within the cluster. 

![3svr_gr_cluster_ip_address_map.png]({{site.baseurl}}/images/3svr_gr_cluster_ip_address_map.png){: .align-center}
Table.1. Geo Redundant Control Plane Cluster IP Addressing
{: .text-center}

