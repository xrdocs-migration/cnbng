---
published: true
date: '2024-02-01'
title: cnBNG CP (AIO) Single VM Deployment Guide - Any VM Environment
position: top
tags:
  - cnbng
  - cse
excerpt: >-
  Learn how to deploy cnBNG Control Plane in a pre-deployed VM in any
  environment
author: Gurpreet Dhaliwal
---
{% include toc %}

cnBNG Control Plane deployment in single VM in any NFVI environment is called as cnBNG CP AIO Manual Deployment. In this deployment cnBNG Control Plane is deployed in a single customized Ubuntu VM. This Ubuntu VM is pre-deployed using SMI base iso image and hence the deployment is called as semi automated or manual. Following are included in this deployment:
- SMI cluster (Cisco CaaS)
- CEE application (Application Infrastructure for Monitoring and Alerting)
- cnBNG Control Plane application (Control Plane application for Cisco CUPS BNG)
    
This is to be noted that only SMI Ubuntu VM deployment in NFVI environment is manual, rest of the process to deploy SMI, CEE and cnBNG Control Plane is fully automated through SMI Deployer. 

For cnBNG CP deployment following steps are followed:
1. Inception VM and SMI Deployer deployment
1. cnBNG CP VM deployment using SMI base ISO and OS customization
1. cnBNG CP deployment using SMI Deployer

## Networking

![aio-networking1.png]({{site.baseurl}}/images/aio-networking1.png)

## Prerequisites: 
- Inception VM running SMI Deployer

## Step 1: Deploying Inception VM and installing SMI Deployer

Refer to [Inception VM and SMI Deployer deployment guide](https://xrdocs.io/cnbng/tutorials/inception-server-deployment-guide/).

## Step 2: cnBNG CP VM deployment using SMI base ISO and OS customization (Manual)

cnBNG CP VM can be deployed using standard VM deployment procedure in a given NFVI environement using SMI base ISO file. 

Following are the manual steps to deploy cnBNG CP VM in VMWare vCenter. Procedure to deploy the VM may differ based on the NFVI environment. 

1. Download the SMI Base ISO file and copy the file to the VM Datastore
1. In the vCenter, select "Create a New Virtual Machine"
1. Specify name of the VM and select the Datacenter
1. Next select the host for the VM
1. Select the datastore 
1. Select compatibility (ESXi 6.7 or later)
1. Select guest OS: Guest Family- Linux, Guest OS Version- Ubuntu Linux (64-bit)
1. Customize Hardware (sizing and networking may vary on deployments):
	1. vCPU: 8, Memory: 16GB, New Hard disk: 100Gb
	1. Under Network: select management network ("VM Network" in most cases
	1. Click New CD/DVD Drive and do the following:
		1. Select Datastore ISO file option to boot from the SMI Base .iso file. Browse to the location of the .iso file on the datastore set in Step 1.
		1. In the Device Status field, select Connect at power on checkbox.
1. After the VM boots up: login to the VM (user: cloud-user, password: Cisco_123). You will be prompted to change the password immediately
1. Now setup Networking by editing /etc/netplan/50-cloud-init.yaml file. Here is a sample file config:
```
	network:
	    ethernets:
	        eno1:
	            dhcp4: true
	        enp0s3:
	            dhcp4: true
	        ens160:
	            dhcp4: false
	            addresses:
	               - 192.168.107.166/25
	            gateway4: 192.168.107.129
	            nameservers:
	                search: [cisco.com]
	                addresses: [72.163.128.140]
	        ens3:
	            dhcp4: true
	        eth0:
	            dhcp4: true
	    version: 2
```

**Note**: If interface is not shown as ens160, search for the right interface using ifconfig -a command. Generally lower ens number is the first NIC attached to the VM, and higher number is the last.
{: .notice--info}

### OS customization

1. SSH login to cnBNG CP VM which is deployed in Step-2
1. Now, change the hostname of the VM to: cnbng-cp-vm, using:
```
sudo hostnamectl set-hostname cnbng-cp-vm
```
1. Logout of the VM and login again to see hostname changes are reflected
1. Make the hostname persistent even after reload by making sure that "preserve_hostname" is set as true in file /etc/cloud/cloud.cfg. If not present already in the file add below statement:
```
preserve_hostname: true
```
1. Change default VM hostname for 127.0.1.1 to "cnbng-cp-vm" in /etc/hosts file
```
127.0.1.1 cnbng-cp-vm cnbng-cp-vm
```
1. Reboot VM and verify that the hostname is persistent after reboot of cnBNG CP VM
1. SSH Key Generation
	1. SSH Login to Inception VM
	1. Generate SSH key using below command. Press enter for anything it asks as input:
```
	ssh-keygen -t rsa
```
	1. Run ssh-copy-id command, which will copy ssh keys to Base ISO Ubuntu Image for cnBNG CP AIO e.g.
```
	ssh-copy-id cloud-user@192.168.107.166
```
	1. Make sure you can login to cnBNG CP VM from Inception VM without password.    
```
	ssh cloud-user@192.168.107.166
```

**Note**: "ssh-copy-id" may not work in latest SMI ISO images. If ssh-copy-id doesnot work then manually copy public key from file /home/cloud-user/.ssh/id_rsa.pub to cnBNG CP VM file /home/cloud-user/.ssh/authorized_keys. Make sure you remove any line breaks from the key. 
{: .notice--info} 

**Warning**: Proceed to next step only if passwordless access to cnBNG CP VM is working from Inception VM.
{: .warning--info} 

## Step 3: cnBNG CP deployment using SMI Deployer

1. Login to Inception VM
1. Login to SMI Deployer running on Inception VM, using below ssh command:
```
ssh admin@localhost -p 2022
```
1. Create environment configs:
	```
	environments manual
		manual
	exit
	```
1. Create Cluster configuration as below with your cnBNG CP VM SSH IP. Change ntp server "clock.cisco.com" to the one available in your lab.

    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    clusters cnbng
     environment manual
     addons ingress bind-ip-address <mark>your-cnbng-cp-vm-ip</mark>
     addons ingress enabled
     addons istio enabled
     configuration master-virtual-ip <mark>your-cnbng-cp-vm-ip</mark>
     configuration master-virtual-ip-interface ens160
     configuration pod-subnet    192.202.0.0/16
     configuration allow-insecure-registry true
     node-defaults initial-boot default-user cloud-user
     node-defaults k8s ssh-username cloud-user
     node-defaults os ntp enabled
     node-defaults os ntp servers <mark>clock.cisco.com</mark>
    exit
    </code>
    </pre>
    </div>
    
1. Configure Inception VM's ssh private key from file /home/cloud-user/.ssh/id_rsa by replace line breaks with "\n", under the cluster config using "node-defaults k8s ssh-connection-private-key". Pay special attention to how the private key is configured under cluster.
```
clusters cnbng
	node-defaults k8s ssh-connection-private-key "-----BEGIN PRIVATE KEY-----\n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\nvEouTDje0Xa3BjWNKh7yntayKA==\n-----END PRIVATE KEY-----"
```
***Here is the original Private Key from file /home/cloud-user/.ssh/id_rsa for your reference:***
```
-----BEGIN PRIVATE KEY-----
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
vEouTDje0Xa3BjWNKh7yntayKA==
-----END PRIVATE KEY-----
```
1. Configure Inception VM's public key from file /home/cloud-user/.ssh/id_rsa.pub, under cluster config using "node-defaults initial-boot default-user-ssh-public-key". Remove any line breaks (if any) in the public key because of copying method: 
```
clusters cnbng
  node-defaults initial-boot default-user-ssh-public-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQClLg/u9ApqA6/NbVUangJj6yMqOZK87/vuFfi2cvL/OOMu/NC1wfaGodlBHgkwtc3NLgvZKa83z4gj26KHzKzIABeSiN18v41bPG2cZSJB8FcHwaXKq00uFbF2tW79GET4p4Q6MoQZE4QSfqynl80WZ+Q8/Di8EgpkEQthp0EHOmMgdn+j4/dHorwwTLw9Ac9lHN+s+OPO1jWUJCELB2gEcbaPQ9n2fFHHZteNgzBYyfUM5MQdZQAT4FXYaWzlTG1XYwSIP0+JapK/0qgC7X06BXuZXpcW5stoFpjDBQatHx74IJQ3ynyMs1IWYgFIL1zurNErXqEJUKkSvFzR3AQTSdr2BgLC1QeTqjNPHzZ9AucO0rZdyy8bDRjV50yt7gRgVOK7b2NoH9B2w8Jgqr9OJcE6GqRCW94EZ2S7ZGzB0UYz0w+S5oNcILKHFfFuCgH234f7LBS3NDIjXfUOUHdadAbSWvGxmXCwrxSG3zxM4vMRcx9hrtpFqCay6gAeGOU= cloud-user@cl-ams-inception"
```
***Here is the original Public Key from file /home/cloud-user/.ssh/id_rsa.pub for your reference:***
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQClLg/u9ApqA6/NbVUangJj6yMqOZK87/vuFfi2cvL/OOMu/NC1wfaGodlBHgkwtc3NLgvZKa83z4gj26KHzKzIABeSiN18v41bPG2cZSJB8FcHwaXKq00uFbF2tW79GET4p4Q6MoQZE4QSfqynl80WZ+Q8/Di8EgpkEQthp0EHOmMgdn+j4/dHorwwTLw9Ac9lHN+s+O
PO1jWUJCELB2gEcbaPQ9n2fFHHZteNgzBYyfUM5MQdZQAT4FXYaWzlTG1XYwSIP0+JapK/0qgC7X06BXuZXpcW5stoFpjDBQatHx74IJQ3ynyMs1IWYgFIL1zurNErXqEJUKkSvFzR3AQTSdr2BgLC1QeTqjNPHzZ9AucO0rZdyy8bDRjV50yt7gRgVOK7b2NoH9B2w8Jgqr9OJcE6GqRCW94EZ2S7ZGzB0UYz0w+S5oNc
ILKHFfFuCgH234f7LBS3NDIjXfUOUHdadAbSWvGxmXCwrxSG3zxM4vMRcx9hrtpFqCay6gAeGOU= cloud-user@cl-ams-inception
```
1. Create AIO Node config:
  
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    clusters cnbng
     nodes cp-vm
      k8s node-type master
      k8s ssh-ip <mark>your-cnbng-cp-vm-ip</mark>
      k8s node-labels disktype ssd
      exit
      k8s node-labels smi.cisco.com/node-type oam
      exit
      initial-boot default-user cloud-user
      initial-boot default-user-password <mark><mark>your-cnbng-cp-vm-ssh-password</mark></mark>
     exit
    exit
    </code>
    </pre>
    </div>
  
1. cnBNG software is available as a tarball and it can be hosted on local http server for offline deployment. In this step we configure the software repository location for tarball. We setup software cnf for both cnBNG CP and CEE. URL and SHA256 depends on the version of the image and the url location, so these two could change for your deployment
```
software cnf bng
  url             http://192.168.107.148/images/CP/cp_30sep21/bng/bng.2021.04.m0.i74.tar
  allow-dev-image true
  sha256          e36b5ff86f35508539a8c8c8614ea227e67f97cf94830a8cee44fe0d2234dc1c
  description     bng-products
exit
software cnf cee
  url         http://192.168.107.148/images/CP/cp_30sep21/cee-2020.02.6.i04.tar
  sha256      b5040e9ad711ef743168bf382317a89e47138163765642c432eb5b80d7881f57
  description cee-products
exit
```
***Note: You can generate sha256 for images using sha256sum.***

1. Setup Ops Center configuration inside cluster configuration and commit
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    clusters cnbng
     ops-centers bng bng
      repository-local        bng
      sync-default-repository true
      netconf-ip              <mark>your-cnbng-cp-vm-ip</mark>
      netconf-port            3022
      ssh-ip                  <mark>your-cnbng-cp-vm-ip</mark>
      ssh-port                2024
      ingress-hostname        <mark>your-cnbng-cp-vm-ip</mark>.nip.io
      initial-boot-parameters use-volume-claims true
      initial-boot-parameters first-boot-password <mark>your-password-to-connect-bng-ops-center</mark>
      initial-boot-parameters auto-deploy false
      initial-boot-parameters single-node true
     exit
     ops-centers cee global
      repository-local        cee
      sync-default-repository true
      netconf-ip              <mark>your-cnbng-cp-vm-ip</mark>
      netconf-port            3024
      ssh-ip                  <mark>your-cnbng-cp-vm-ip</mark>
      ssh-port                2023
      ingress-hostname        <mark>your-cnbng-cp-vm-ip</mark>.nip.io
      initial-boot-parameters use-volume-claims true
      initial-boot-parameters first-boot-password <mark>your-password-to-connect-cee-ops-center</mark>
      initial-boot-parameters auto-deploy true
      initial-boot-parameters single-node true
     exit
    exit
    </code>
    </pre>
    </div>
  
1. Deploy cnBNG CP cluster using cluster sync command:
  ```
  clusters cnbng actions sync run
  ```	

1. Monitor sync using monitor command:
  ```
  monitor sync-logs cnbng
  ```
  
## Verifications

- After successful deployment of the cluster, we can check kubernetes PODs running in the cluster using below command.

  ```
  cloud-user@cnbng-cp-vm:~$ kubectl get pods -A
  NAMESPACE           NAME                                                 READY   STATUS    RESTARTS   AGE
  bng-bng             documentation-95c8f45d9-8g4qd                        1/1     Running   0          5m
  bng-bng             ops-center-bng-bng-ops-center-77fb6479fc-dtvt2       5/5     Running   0          5m
  bng-bng             smart-agent-bng-bng-ops-center-8d9fffbfb-5cdqm       1/1     Running   1          5m
  cee-global          alert-logger-7c6d5b6596-jrrx6                        1/1     Running   0          5m
  cee-global          alert-router-549c8fb66c-4shz6                        1/1     Running   0          5m
  cee-global          alertmanager-0                                       1/1     Running   0          5m
  cee-global          blackbox-exporter-n9mzx                              1/1     Running   0          5m
  cee-global          bulk-stats-0                                         3/3     Running   0          5m
  cee-global          cee-global-product-documentation-864c8fb66b-rm44g    2/2     Running   0          5m
  cee-global          core-retriever-bntcb                                 2/2     Running   0          5m
  cee-global          documentation-684cbb8cbc-95qvj                       1/1     Running   0          5m
  cee-global          grafana-6f54c8cc5f-xpvj5                             6/6     Running   0          5m
  cee-global          grafana-dashboard-metrics-7764c5f8f4-kgthj           1/1     Running   0          5m
  cee-global          kube-state-metrics-79bdbd9db7-jmxxx                  1/1     Running   0          5m
  cee-global          logs-retriever-b7l5v                                 1/1     Running   0          5m
  cee-global          node-exporter-rnhwr                                  1/1     Running   0          5m
  cee-global          ops-center-cee-global-ops-center-5bbdb84597-8rtk5    5/5     Running   0          5m
  cee-global          path-provisioner-l89w6                               1/1     Running   0          5m
  cee-global          pgpool-859f9d7d89-mtks5                              1/1     Running   0          5m
  cee-global          pgpool-859f9d7d89-rwsmk                              1/1     Running   0          5m
  cee-global          postgres-0                                           1/1     Running   0          5m
  cee-global          postgres-1                                           1/1     Running   0          5m
  cee-global          postgres-2                                           1/1     Running   0          5m
  cee-global          prometheus-hi-res-0                                  4/4     Running   0          5m
  cee-global          prometheus-rules-65ccd95b6c-29n6j                    1/1     Running   0          5m
  cee-global          prometheus-scrapeconfigs-synch-85fdc7ccbf-jd54g      1/1     Running   0          5m
  cee-global          pv-manager-9449fc649-h97f4                           1/1     Running   0          5m
  cee-global          pv-provisioner-6775997cc5-hjzf6                      1/1     Running   0          5m
  cee-global          restart-kubelet-75f7d                                1/1     Running   0          5m
  cee-global          show-tac-manager-5cbb8d589b-rpzv9                    2/2     Running   0          5m
  cee-global          smart-agent-cee-global-ops-center-69f7b8d9d5-ptrct   1/1     Running   1          5m
  cee-global          thanos-query-frontend-hi-res-54bbbbbcb6-phbcq        1/1     Running   0          5m
  cee-global          thanos-query-hi-res-6776585f99-bfzx9                 2/2     Running   0          5m
  istio-system        istiod-cf97f695b-6qqzp                               1/1     Running   0          5m
  kube-system         calico-kube-controllers-5c9bfb6cf-jnrc8              1/1     Running   0          5m
  kube-system         calico-node-ptnjs                                    1/1     Running   0          5m
  kube-system         cluster-cert-maintainer-7f879cf5cf-88xkn             1/1     Running   0          5m
  kube-system         coredns-558bd4d5db-kkhd2                             1/1     Running   0          5m
  kube-system         coredns-558bd4d5db-q5lct                             1/1     Running   0          5m
  kube-system         etcd-pod2-cnbng-cp                                   1/1     Running   0          5m
  kube-system         kube-apiserver-pod2-cnbng-cp                         1/1     Running   0          5m
  kube-system         kube-controller-manager-pod2-cnbng-cp                1/1     Running   0          5m
  kube-system         kube-proxy-lp68c                                     1/1     Running   0          5m
  kube-system         kube-scheduler-pod2-cnbng-cp                         1/1     Running   0          5m
  kube-system         maintainer-sgrs7                                     1/1     Running   0          5m
  nginx-ingress       nginx-ingress-controller-b5857775-slgkh              1/1     Running   0          5m
  nginx-ingress       nginx-ingress-default-backend-86496666db-5jgcz       1/1     Running   0          5m
  registry            charts-bng-2021-04-m0-i74-0                          1/1     Running   0          5m
  registry            charts-cee-2020-02-6-i04-0                           1/1     Running   0          5m
  registry            registry-bng-2021-04-m0-i74-0                        1/1     Running   0          5m
  registry            registry-cee-2020-02-6-i04-0                         1/1     Running   0          5m
  registry            software-unpacker-0                                  1/1     Running   0          5m
  smi-certs           ss-cert-provisioner-5d764d5667-27d55                 1/1     Running   0          5m
  smi-secure-access   secure-access-controller-k62z5                       1/1     Running   0          5m
  smi-vips            keepalived-5l789                                     3/3     Running   0          5m
  ```

- Check Grafana ingress and try logging to it (username: admin, password: your-password-to-connect-cee-ops-center)
  
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cloud-user@cnbng-cp-vm:~$ kubectl get ingress -A
    NAMESPACE    NAME                                       CLASS    HOSTS                                                          ADDRESS           PORTS     AGE
    bng-bng      cli-ingress-bng-bng-ops-center             &lt;none&gt;   cli.bng-bng-ops-center.192.168.107.150.nip.io                  192.168.107.150   80, 443   5m
    bng-bng      documentation-ingress                      &lt;none&gt;   documentation.bng-bng-ops-center.192.168.107.150.nip.io        192.168.107.150   80, 443   5m
    bng-bng      restconf-ingress-bng-bng-ops-center        &lt;none&gt;   restconf.bng-bng-ops-center.192.168.107.150.nip.io             192.168.107.150   80, 443   5m
    cee-global   cee-global-product-documentation-ingress   &lt;none&gt;   docs.cee-global-product-documentation.192.168.107.150.nip.io   192.168.107.150   80, 443   5m
    cee-global   cli-ingress-cee-global-ops-center          &lt;none&gt;   cli.cee-global-ops-center.192.168.107.150.nip.io               192.168.107.150   80, 443   5m
    cee-global   documentation-ingress                      &lt;none&gt;   documentation.cee-global-ops-center.192.168.107.150.nip.io     192.168.107.150   80, 443   5m
    cee-global   grafana-ingress                            &lt;none&gt;   <mark>grafana.192.168.107.150.nip.io</mark>                                 192.168.107.150   80, 443   5m
    cee-global   prometheus-hi-res                          &lt;none&gt;   prometheus-hi-res.192.168.107.150.nip.io                       192.168.107.150   80, 443   5m
    cee-global   restconf-ingress-cee-global-ops-center     &lt;none&gt;   restconf.cee-global-ops-center.192.168.107.150.nip.io          192.168.107.150   80, 443   5m
    cee-global   show-tac-manager-ingress                   &lt;none&gt;   show-tac-manager.192.168.107.150.nip.io                        192.168.107.150   80, 443   5m
    registry     charts-ingress                             &lt;none&gt;   charts.192.168.107.150.nip.io                                  192.168.107.150   80, 443   5m
    registry     registry-ingress                           &lt;none&gt;   docker.192.168.107.150.nip.io                                  192.168.107.150   80, 443   5m
    </code>
    </pre>
    </div>

- We can login to Grafana GUI from Chrome/ any browser @URL: https://grafana.your-cnbng-cp-vm-ip.nip.io/

	![grafana-login1.png]({{site.baseurl}}/images/grafana-login1.png)

- SSH to cnBNG CP Ops Center at port 2024
  
  ```
  cloud-user@inception:~$ ssh admin@your-cnbng-cp-vm-ip -p 2024
  admin@192.168.107.150's password: 

        Welcome to the bng CLI on pod2/bng
        Copyright © 2016-2020, Cisco Systems, Inc.
        All rights reserved.

  User admin last logged in 2021-11-19T04:12:32.912093+00:00, to ops-center-bng-bng-ops-center-77fb6479fc-dtvt2, from 192.168.107.150 using cli-ssh
  admin connected from 192.168.107.150 using ssh on ops-center-bng-bng-ops-center-77fb6479fc-dtvt2
  [cnbng/bng] bng# 
  [cnbng/bng] bng# 
  ```
  
- We can also test Netconf Interface availability of cnBNG Ops Center using ssh
  
  ```
  cloud-user@inception:~$ ssh admin@your-cnbng-cp-vm-ip -p 3022 -s netconf    
  admin@192.168.107.150's password: 
  <?xml version="1.0" encoding="UTF-8"?>
  <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
  <capability>urn:ietf:params:netconf:base:1.0</capability>
  <capability>urn:ietf:params:netconf:base:1.1</capability>
  <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
  <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:rollback-on-error:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:url:1.0?scheme=ftp,sftp,file</capability>
  <capability>urn:ietf:params:netconf:capability:validate:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
  <capability>urn:ietf:params:netconf:capability:xpath:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:interleave:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:partial-lock:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&amp;also-supported=report-all-tagged,report-all</capability>
  <capability>urn:ietf:params:netconf:capability:yang-library:1.0?revision=2019-01-04&amp;module-set-id=a6803bdbd5c766b47137ad86700fff0a</capability>
  <capability>urn:ietf:params:netconf:capability:yang-library:1.1?revision=2019-01-04&amp;content-id=a6803bdbd5c766b47137ad86700fff0a</capability>
  <capability>http://tail-f.com/ns/netconf/actions/1.0</capability>
  <capability>http://cisco.com/cisco-bng-ipam?module=cisco-bng-ipam&amp;revision=2020-01-24</capability>
  <capability>http://cisco.com/cisco-cn-ipam?module=cisco-cn-ipam&amp;revision=2021-02-21</capability>
  <capability>http://cisco.com/cisco-exec-ipam?module=cisco-exec-ipam&amp;revision=2021-06-01</capability>
  <capability>http://cisco.com/cisco-mobile-nf-tls?module=cisco-mobile-nf-tls&amp;revision=2020-06-24</capability>
  <capability>http://cisco.com/cisco-smi-etcd?module=cisco-smi-etcd&amp;revision=2021-09-15</capability>
  <capability>http://tail-f.com/cisco-mobile-common?module=tailf-mobile-common&amp;revision=2019-04-25</capability>
  <capability>http://tail-f.com/cisco-mobile-product?module=tailf-cisco-mobile-product&amp;revision=2018-06-06</capability>
  <capability>http://tail-f.com/ns/aaa/1.1?module=tailf-aaa&amp;revision=2018-09-12</capability>
  <capability>http://tail-f.com/ns/common/query?module=tailf-common-query&amp;revision=2017-12-15</capability>
  <capability>http://tail-f.com/ns/confd-progress?module=tailf-confd-progress&amp;revision=2020-06-29</capability>
  <capability>http://tail-f.com/ns/kicker?module=tailf-kicker&amp;revision=2020-11-26</capability>
  <capability>http://tail-f.com/ns/netconf/query?module=tailf-netconf-query&amp;revision=2017-01-06</capability>
  <capability>http://tail-f.com/ns/webui?module=tailf-webui&amp;revision=2013-03-07</capability>
  <capability>http://tail-f.com/yang/acm?module=tailf-acm&amp;revision=2013-03-07</capability>
  <capability>http://tail-f.com/yang/common?module=tailf-common&amp;revision=2020-11-26</capability>
  <capability>http://tail-f.com/yang/common-monitoring?module=tailf-common-monitoring&amp;revision=2019-04-09</capability>
  <capability>http://tail-f.com/yang/confd-monitoring?module=tailf-confd-monitoring&amp;revision=2019-10-30</capability>
  <capability>http://tail-f.com/yang/last-login?module=tailf-last-login&amp;revision=2019-11-21</capability>
  <capability>http://tail-f.com/yang/netconf-monitoring?module=tailf-netconf-monitoring&amp;revision=2019-03-28</capability>
  <capability>http://tail-f.com/yang/xsd-types?module=tailf-xsd-types&amp;revision=2017-11-20</capability>
  <capability>urn:ietf:params:xml:ns:netconf:base:1.0?module=ietf-netconf&amp;revision=2011-06-01&amp;features=confirmed-commit,candidate,rollback-on-error,validate,xpath,url</capability>
  <capability>urn:ietf:params:xml:ns:netconf:partial-lock:1.0?module=ietf-netconf-partial-lock&amp;revision=2009-10-19</capability>
  <capability>urn:ietf:params:xml:ns:yang:iana-crypt-hash?module=iana-crypt-hash&amp;revision=2014-08-06&amp;features=crypt-hash-sha-512,crypt-hash-sha-256,crypt-hash-md5</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-inet-types?module=ietf-inet-types&amp;revision=2013-07-15</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-netconf-acm?module=ietf-netconf-acm&amp;revision=2018-02-14</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring?module=ietf-netconf-monitoring&amp;revision=2010-10-04</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-netconf-notifications?module=ietf-netconf-notifications&amp;revision=2012-02-06</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-netconf-with-defaults?module=ietf-netconf-with-defaults&amp;revision=2011-06-01</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-restconf-monitoring?module=ietf-restconf-monitoring&amp;revision=2017-01-26</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-x509-cert-to-name?module=ietf-x509-cert-to-name&amp;revision=2014-12-10</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-yang-metadata?module=ietf-yang-metadata&amp;revision=2016-08-05</capability>
  <capability>urn:ietf:params:xml:ns:yang:ietf-yang-types?module=ietf-yang-types&amp;revision=2013-07-15</capability>
  </capabilities>
  <session-id>171</session-id></hello>]]>]]>
  ```
  
## Initial cnBNG CP Configurations

If you have deployed cnBNG CP a fresh then most probably initial cnBNG CP configuration is not applied on Ops Center. Follow below steps to apply initial configuratuon to cnBNG CP Ops Center

- SSH login to cnBNG CP Ops Center CLI

  ```
  cloud-user@inception:~$ ssh admin@your-cnbng-cp-vm-ip -p 2024         
  Warning: Permanently added '[192.168.107.150]:2024' (RSA) to the list of known hosts.
  admin@192.168.107.150's password: 

        Welcome to the bng CLI on pod100/bng
        Copyright © 2016-2020, Cisco Systems, Inc.
        All rights reserved.

  User admin last logged in 2021-12-01T11:42:45.247257+00:00, to ops-center-bng-bng-ops-center-5666d4cb6-dj7sv, from 192.168.107.150 using cli-ssh
  admin connected from 192.168.107.150 using ssh on ops-center-bng-bng-ops-center-5666d4cb6-dj7sv
  [cnbng/bng] bng# 
  ```

- Change to config mode in Ops Center

  ```
  [cnbng/bng] bng# config
  Entering configuration mode terminal
  [cnbng/bng] bng(config)# 
  ```

- Apply following initial configuration. With changes to "endpoint radius" and "udp proxy" configs. Both "endpoint radius" and "udp-proxy" should use IP of cnBNG CP VM.

  <div class="highlighter-rouge">
  <pre class="highlight">
  <code>
  cdl node-type session
  cdl zookeeper replica 1
  cdl logging default-log-level error
  cdl datastore session
   slice-names [ 1 ]
   endpoint replica 1
   endpoint settings slot-timeout-ms 750
   index replica 1
   index map    1
   index write-factor 1
   slot replica 1
   slot map     1
   slot write-factor 1
   slot notification limit 300
  exit
  cdl kafka replica 1
  instance instance-id 1
   endpoint sm
   exit
   endpoint nodemgr
   exit
   endpoint n4-protocol
    retransmission timeout 0 max-retry 0
   exit
   endpoint dhcp
   exit
   endpoint pppoe
   exit
   endpoint radius
  !! Change this IP to your cnBNG CP VM IP
     <mark>vip-ip your-cnbng-cp-vm-ip</mark>
    interface coa-nas
     sla response 140000
  !! Change this IP to your cnBNG CP VM IP
     <mark>vip-ip your-cnbng-cp-vm-ip vip-port 2000</mark>
    exit
   exit
   endpoint udp-proxy
  !! Change this IP to your cnBNG CP VM IP
    <mark>vip-ip your-cnbng-cp-vm-ip</mark>
   exit
  exit
  deployment
   app-name     BNG
   cluster-name cnbng
   dc-name      DC
   model        small
  exit
  k8 bng
   etcd-endpoint      etcd:2379
   datastore-endpoint datastore-ep-session:8882
   tracing
    enable
    enable-trace-percent 30
    append-messages      true
    endpoint             jaeger-collector:9411
   exit
  exit
  instances instance 1
   system-id  DC
   cluster-id cnbng
   slice-name 1
  exit
  local-instance instance 1
  </code>
  </pre>
  </div>

- Configure "system mode running" and commit

  <div class="highlighter-rouge">
  <pre class="highlight">
  <code>
  [cnbng/bng] bng(config)# <mark>system mode running </mark>   
  [cnbng/bng] bng(config)# commit
  Commit complete.
  [cnbng/bng] bng(config)# 
  Message from confd-api-manager at 2021-12-01 12:36:05...
  Helm update is STARTING.  Trigger for update is STARTUP. 
  Message from confd-api-manager at 2021-12-01 12:36:08...
  Helm update is SUCCESS.  Trigger for update is STARTUP.
  </code>
  </pre>
  </div>

- Wait for system to deploy all PODs. Verify that all PODs are deployed for cnBNG. Four PODs will be in <span style="background-color: #FDD7E4">Init</span> state at this moment, which is ok.

  <div class="highlighter-rouge">
  <pre class="highlight">
  <code>
  cloud-user@cnbng-cp-vm:~$ kubectl get pods -n bng-bng
  NAME                                                   READY   STATUS     RESTARTS   AGE
  bng-dhcp-n0-0                                          0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
  bng-n4-protocol-n0-0                                   2/2     Running    0          2m44s
  bng-nodemgr-n0-0                                       0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
  bng-pppoe-n0-0                                         0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
  bng-sm-n0-0                                            0/2     <span style="background-color: #FDD7E4">Init:0/1</span>   0          2m45s
  cache-pod-0                                            1/1     Running    0          2m44s
  cache-pod-1                                            0/1     Running    0          16s
  cdl-ep-session-c1-d0-868f578d69-8n2rw                  1/1     Running    0          2m46s
  cdl-index-session-c1-m1-0                              1/1     Running    0          2m46s
  cdl-slot-session-c1-m1-0                               1/1     Running    0          2m46s
  documentation-5857c6db9f-82lg6                         1/1     Running    0          46h
  etcd-bng-bng-etcd-cluster-0                            2/2     Running    0          2m45s
  etcd-bng-bng-etcd-cluster-1                            2/2     Running    0          2m45s
  etcd-bng-bng-etcd-cluster-2                            2/2     Running    0          2m45s
  georeplication-pod-0                                   1/1     Running    1          2m44s
  grafana-dashboard-app-infra-bng-bng-64b5f7c8b4-rsg5v   1/1     Running    0          2m45s
  grafana-dashboard-cdl-bng-bng-66947dbb96-p6c8n         1/1     Running    0          2m46s
  grafana-dashboard-cnbng-749978f9cb-txfnd               1/1     Running    0          2m45s
  grafana-dashboard-etcd-bng-bng-76b5f7b796-whlch        1/1     Running    0          2m45s
  kafka-0                                                2/2     Running    0          2m46s
  oam-pod-0                                              2/2     Running    2          2m46s
  ops-center-bng-bng-ops-center-5666d4cb6-dj7sv          5/5     Running    0          46h
  prometheus-rules-cdl-56457cfd8-tj2d4                   1/1     Running    0          2m46s
  prometheus-rules-etcd-5796974fb5-t9w6m                 1/1     Running    0          2m45s
  radius-ep-n0-0                                         2/2     Running    1          2m45s
  smart-agent-bng-bng-ops-center-54bcbf5576-tc4p9        1/1     Running    1          46h
  udp-proxy-0                                            1/1     Running    0          2m45s
  zookeeper-0                                            1/1     Running    0          2m46s
  </code>
  </pre>
  </div>
