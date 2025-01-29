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

1. Login to local OCP cluster console as admin and follow below steps.

### Node Labels

1. Add following labels to the node. These labels are required for cnBNG POD deployment/ placement. Add labels by selecting Nodes under Compute from side panel.
  ```
  smi.cisco.com/node-type=oam
  disktype=ssd
  ```
  ![Screenshot 2025-01-28 at 3.57.02 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-28 at 3.57.02 PM.png)
  
  ![Screenshot 2025-01-28 at 3.57.06 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-28 at 3.57.06 PM.png)

### Creating Namespaces

1. Create two namespaces "ceeocp" and "bngocp" by selecting Namespaces under Administration from side panel
	![Screenshot 2025-01-29 at 12.02.45 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.02.45 PM.png)
    
    <table style="width:100%" border = "2">
    <tr>
      <td><img src="/cnbng/images/Screenshot 2025-01-29 at 12.03.17 PM.png"></td>
      <td><img src="/cnbng/images/Screenshot 2025-01-29 at 12.03.45 PM.png"></td>
    </tr>
    </table>

### Creating Secrets

Secrets are required to pull Helm Charts and Docker Images for cnBNG Control Plane and CEE application. In this step we will create helm secret and docker image pull seceret for both namespaces "ceeocp" and "bngocp".

1. Switch to the Developer mode from Administrator mode by selecting it from Side Panel
	![Screenshot 2025-01-29 at 12.14.27 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.14.27 PM.png)

1. In Developer mode, select "Secrets" from side panel. Now click on "Create->From YAML" under project ceeocp.
	![Screenshot 2025-01-29 at 12.28.11 PM.jpeg]({{site.baseurl}}/images/Screenshot 2025-01-29 at 12.28.11 PM.jpeg)

1. Copy paste below yaml
  ```
  apiVersion: v1
  kind: Secret
  metadata:
      name: cnbng-helm-repo-secret
  stringData:
      username: your-helm-repo-username
      password: your-helm-repo-password
  ```

1. Click again on "Create->Image pull secret" for "ceeocp" project. Fill form data as per below information:

    **Secret Name:** cnbng-docker-repo-secret

    **Authentication type:** Image registry credentials

    **Registry server address:** your-docker-hub-url

    **Username:** your-username-for-dockerhub

    **Password:** your-password-to-access-dockerhub

1. Similarly create Helm secret and docker secret for bngocp project, using same yaml and form data, as was created for ceeocp project.

### Creating Helm Repositories

Now we will create Helm repositories for CEE and cnBNG Control Plane applications. 

1. Select "Create->Repository" by following options as highlighted below.
  ![Screenshot 2025-01-29 at 2.09.58 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 2.09.58 PM.png)

1. Select YAML view and copy paste below YAML in place of default YAML provided.
	```
    apiVersion: helm.openshift.io/v1beta1
    kind: ProjectHelmChartRepository
    metadata:
      name: ceeocp
      namespace: ceeocp
    spec:
      name: ceeocp
      connectionConfig:
        url: >-      
          https://your-helm-chart-repo-url-for-cee
        basicAuthConfig:
          name: cnbng-helm-repo-secret
    ```
    **Note** Make sure you have selected "ceeocp" as the Project
	{: .notice--info}
    
1. Similarly create Repository for "bngocp" Project using below YAML.
	```
    apiVersion: helm.openshift.io/v1beta1
    kind: ProjectHelmChartRepository
    metadata:
      name: bngocp
      namespace: bngocp
    spec:
      name: bngocp
      connectionConfig:
        url: >-
          https://your-helm-chart-repo-url-for-cnbng-control-plane
        basicAuthConfig:
          name: cnbng-helm-repo-secret
	```
    
With this we complete preparation step for OCP. 

## Deploying CEE

In this step we will deploy CEE PODs by first deploying CEE Ops Center POD and then doing necessary configurations on CEE Ops Center to deploy rest of CEE PODs.

### Deploying CEE Ops Center POD

Now since we have got the helm repositories created, we can proceed to create helm release and deploy CEE Ops Center POD.

1. Click on "Create->Helm Release" to create helm release from the charts.
	
    ![Screenshot 2025-01-29 at 2.36.32 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 2.36.32 PM.png)
	
1. After release is created click on Ops Center POD chart for ceeocp.

	![Screenshot 2025-01-29 at 2.42.28 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 2.42.28 PM.png)
	
1. Click on Create for YAML definitions.

	![Screenshot 2025-01-29 at 2.41.29 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 2.41.29 PM.png)
	
1. Replace default YAML definition with YAML definition below and change "Release name". 
	
    ![Screenshot 2025-01-29 at 2.47.39 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 2.47.39 PM.png)
    
	```
    global:
      caasModel: "openshift"
      imagePullPolicy: IfNotPresent
      imagePullSecrets:
        - name: cnbng-docker-repo-secret
      ingressHostname: your-node-ip-address.nip.io
      istio: false
      networkPolicy:
        enabled: false
      pathBasedIngress: true
      registry: your-dockerhub-url
      singleNode: true
      smiNfMonitoring: true
      smiPlatformMonitoring: true
      useVolumeClaims: true
    ops-center:
      product:
        helm:
          product: cee
          autoDeploy: false
          repository:
            url: https://your-helm-chart-repo-url-for-cee
            name: ceeocp
            accessToken: your-helm-repo-username:your-helm-repo-access-password
    helm-api:
      env:
        - name: DOWNLOAD_CHART_PROXY
          value: "https://your-proxy-url"
    ```
	
1. Once the POD is deployed successfully and is in Running state, click on the POD name "ops-center-cee-ops-center-ams1"
	
	![Screenshot 2025-01-29 at 2.57.06 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 2.57.06 PM.png)
	    
1. Scroll down and verify that all four containers are in Running state.

	![Screenshot 2025-01-29 at 3.01.18 PM.jpeg]({{site.baseurl}}/images/Screenshot 2025-01-29 at 3.01.18 PM.jpeg)

### Configuring CEE Ops Center

1. Click on Terminal for CEE Ops Center POD and connect to confd_cli using command ```bin/confd_cli -u admin```

	![Screenshot 2025-01-29 at 3.45.58 PM.png]({{site.baseurl}}/images/Screenshot 2025-01-29 at 3.45.58 PM.png)
    
	**Note** It will ask to set the passoword when you connect for the first time.
	{: .notice--info}
	
1. Configure following on CEE Ops Center to get all the PODs running for CEE.
	
    ```
    config
    pv-provisioner pv-path /var/data
    k8s name cnbng
    system mode running
    commit
    ```
    
1. Verify that the "system status percent-ready" is 100% using command "show system" before proceeding further. 
	
	<div class="highlighter-rouge">
    <pre class="highlight">
    <code>
	[unknown] cee# show system
    system uuid 3e2b0987-4f8b-40fa-a2c2-0cb2969e7882
    system status deployed true
    system status percent-ready <mark>100.0</mark>
    sustem ons-center renosttory unknown system
    ops-center-debug status false
    system synch running true system synch pending
    </code>
    </pre>
    </div>

With this we complete deployment of CEE application.

## Deploying cnBNG Control Plane

In this step we will deploy cnBNG Control Plane (CP) PODs by first deploying cnBNG CP Ops Center POD and then doing necessary configurations on the Ops Center to deploy rest of the PODs.

### Deploying cnBNG Control Plane Ops Center

