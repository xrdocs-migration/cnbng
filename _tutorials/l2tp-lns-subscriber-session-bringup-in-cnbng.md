---
published: true
date: '2025-05-15 09:25 +0530'
title: L2TP LNS Subscriber Session Bringup in cnBNG
position: top
author: Gurpreet Dhaliwal
tags:
  - cnbng
  - l2tp
  - lns
---
{% include toc %}

## Introduction

In this tutorial we will learn how to bring-up L2TP LNS subscriber session in Cloud Native BNG (cnBNG). We will configure this lab to have dual stack LNS sessions. We will setup radius profile to authenticate LNS subscriber sessions, push plan policies and other attributes.

We will bringup two types of LNS sessions:
- Dual Stack Dynamic Session, where IP addresses (IPv4, IPv6 NA, IPv6 PD) are dynamically assigned through IPAM
- Dual Stack Static Session, where IP addresses (IPv4, IPv6 NA, IPv6 PD) are statically assigned  through Radius

We will apply below features to each session:
- QoS
- ACL
- uRPF
- Framed Route (IPv4 + IPv6)

## Topology
The setup used for this tutorial is shown in figure 1, LNS UP is ASR9k-1. This setup uses Spirent to emulate LAC and client devices. Spirent port 1/4 will be used for LAC and client emulation. 

![lns-topo.png]({{site.baseurl}}/images/lns-topo.png)

## Call Flow for LNS
Following figure shows the call flow for LNS in cnBNG.

![lns-call-flow.png]({{site.baseurl}}/images/lns-call-flow.png)


## Prerequisite
Make sure l2tp-tunnel endpoint is configured in cnBNG CP Ops-Center and the corresponding POD is running for LNS sessions to work.

```
instance instance-id 1
 endpoint l2tp-tunnel
 exit
exit
```

Verify l2tp-tunnel POD is running on K8s Master VM:

```
cisco@cnbng-tme-lab-aio-cp:~$ kubectl get pods -n bng-bng | grep l2tp
bng-l2tp-tunnel-n0-0                                   1/1     Running   1          22h
```

## cnBNG CP Configuration

cnBNG CP Configuration has following constructs/parts for LNS:
- IPAM
- Profile PPPoE
- Profile DHCP (applicable for IPv6 address assignment only)
- Profile AAA
- Profile Radius
- Profile Feature-Template
- Profile L2TP
- Profile Subscriber
- User-Plane

Let's understand each construct in step-by-step manner.

### IPAM
IPAM defines subscriber address pools for IPv4, IPv6 (NA) and IPv6 (PD). These are the pools from which LNS client will get the IP addresses. IPAM assigns addresses dynamically by splitting address pools into smaller chunks and then associating each chunk with a user-plane. The pools get freed up dynamically and re-allocated to different user-planes on need basis. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ipam
 instance 1
  source local
  address-pool pool-Silver
   vrf-name default
   ipv4
    split-size
     per-dp 1024
    exit
    address-range 192.168.0.1 192.168.99.254
   exit
   ipv6
    address-ranges
     split-size
      per-dp 1024
     exit
     address-range 2001:192:168::1 2001:192:168::1:100
    exit
    prefix-ranges
     split-size
      per-dp 1024
     exit
     prefix-range 2008:: length 48
    exit
   exit
  exit
  address-pool pool-Static
   vrf-name default
   static enable
   ipv4
    split-size
     no-split
    exit
    address-range 112.0.0.1 112.0.0.253 default-gateway 112.0.0.254
   exit
   ipv6
    address-ranges
     split-size
      no-split
     exit
     address-range 2001:112::1 2001:112::255
    exit
    prefix-ranges
     split-size
      no-split
     exit
     prefix-range 2001:111:: length 48
    exit
   exit
  exit
 exit
exit
</code>
</pre>
</div>

### Profile PPPoE
This profile is same as the BBA Group which was defined on ASR9k integrated BNG solution. We define service names etc. For this tutorial we will keep it simple and only specify the MTU.

```
profile pppoe ppp1
 mtu 1494
exit
```

### Profile DHCP
Incase of L2TP LNS Dual Stack or IPv6 only subscribers we will be using DHCPv6 server to assign the IPv6 (IANA+IAPD) prefixes to LNS Client. For this example we will have cnBNG CP act as a DHCP server to assign IPv6 addresses to CPE/subscribers. In profile DHCP we define the DHCP server and which IPAM pool to use by default for subscriber. We can use different pools for IPv4, IPv6 (IANA) and IPv6 (IAPD). 

```
profile dhcp dhcp-server1
 ipv4
  mode server
  server
   pool-name   pool-Silver
   dns-servers [ 8.8.8.8 ]
   lease days 1
  exit
 exit
 ipv6
  mode server
  server
   iana-pool-name pool-Silver
   iapd-pool-name pool-Silver
   lease days 1
  exit
 exit
exit
```

**Note**: The definition of IPv4 server profile is not needed for LNS subscribers. For LNS subscribers IPv4 addresses will be assigned by IPCP using IPAM directly.
{: .notice--info}

### Profile AAA
This profile defines the AAA parameters, like which Radius group to be used for authentication/authorization and accounting. In this tutorial we will be using radius group defined as "local" under radius profile for authentication and accounting.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile aaa aaa_pppoe
 authentication
  method-order [ <mark>local</mark> ]
 exit
 accounting
  method-order [ <mark>local</mark> ]
 exit
exit
</code>
</pre>
</div>

### Profile Radius
Under this profile, Radius groups are created.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile server-group <mark>local</mark>
 radius-group <mark>local</mark>
exit

profile radius
 algorithm round-robin
 deadtime  3
 detect-dead-server response-timeout 60
 max-retry 2
 timeout   5
 <mark>!!! Radius server IP and port definitions for auth and acct</mark>
 server <mark>192.168.107.152 1812</mark>
  type   auth
  secret cisco
 exit
 server <mark>192.168.107.152 1813</mark>
  type   acct
  secret cisco
 exit
 attribute
  nas-identifier CISCO-BNG
  <mark>!!! This should be protocol VIP to reach Radius</mark>
  nas-ip         <mark>192.168.107.165</mark>
 exit
 server-group <mark>local</mark>
  server auth <mark>192.168.107.152 1812</mark>
  exit
  server acct <mark>192.168.107.152 1813</mark>
  exit
 exit
exit
<mark>!!! we can also set COA client</mark>
profile <mark>coa</mark>
 client <mark>192.168.107.152</mark>
  server-key cisco
 exit
exit
</code>
</pre>
</div>

### Profile Feature-template
This profile defines subscriber feature template. This is the template which will be applied to dynamic subscriber interface. We also enable service/ session accounting here.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile feature-template ft-pppoe-lns-1
 vrf-name default
 ipv4
  mtu 1500
 exit
 session-accounting
  enable
  aaa-profile       aaa_pppoe
  periodic-interval 60
 exit
 ppp
  authentication [ chap pap ]
  ipcp peer-address-pool pool-Silver !! Pool for IPv4 address assignment using IPCP
  ipcp renegotiation ignore
  ipv6cp renegotiation ignore
  lcp renegotiation ignore
  max-bad-auth   4
  max-failure    5
  timeout absolute 1440
  timeout authentication 5
  timeout retry  4
  keepalive disable
 exit
exit
</code>
</pre>
</div>

### Profile L2TP
This profile defines the l2tp parameters for LNS sessions. L2TP Tunnel source and destination IPs along with authentication and other parameters are defined under this profile.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
profile l2tp lns-up1
 mode           lns
 hostname       lns.cisco.com
 hello-interval 600
 retransmit timeout max 8
 retransmit timeout min 4
 retransmit retries 10
 receive-window 1024
 vrf            default
 authentication
 ip-tos reflect
 tunnel session-limit 640
 tunnel timeout no-session 10
 password       your-tunnel-password
 ipv4 df-bit reflect
 ipv4 source 172.0.0.2 !! User Plane Loopback 0 IP
exit
</code>
</pre>
</div>

### Profile Subscriber
This profile can be attached on per access port level or per user-plane level. This profile for PPPoE defines which dhcp server profile to apply for IPv6 address assignment using DHCPv6, along with feature-template, l2tp-profile and aaa-profile to be used for auth/acct. 

```
profile subscriber subscriber-profile_pppoe-lns-up1
 dhcp-profile               dhcp-server1
 pppoe-profile              ppp1
 session-type               ipv4v6
 l2tp-profile               lns-up1
 activate-feature-templates [ ft-pppoe-lns-1 ]
 event session-activate
  aaa authenticate aaa_pppoe
 exit
exit
```

### User-plane
This construct define the association configs. Peering IP as well as subscriber profile to be attached to user-plane or at port level. In this tutorial we will attach subscriber profile at port level.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
user-plane
 instance 1
  user-plane ASR9k-1
   peer-address ipv4 192.168.107.142
   port-id Bundle-Ether24
    subscriber-profile subscriber-profile_pppoe-lns-up1
   exit
   port-id Bundle-Ether25
    subscriber-profile subscriber-profile_pppoe-lns-up1
   exit
  exit
  user-plane ASR9k-2
   peer-address ipv4 192.168.107.153
  exit
 exit
exit
</code>
</pre>
</div>

## cnBNG UP Configuration

UP Configuration has mainly four constructs for L2TP LNS:

- Association Configuration
- DHCP Configuration
- Access Interface
- Feature definitions: QoS, ACL

### Association Configuration
This is where we define association settings between cnBNG CP and UP. The auto-loopback with "secondary-address-upadte enable" will allow dynamic IP address allocations using IPAM for LNS sessions. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
cnbng-nal location 0/RSP0/CPU0
 hostidentifier ASR9k-1
 up-server ipv4 192.168.107.142 vrf default
 cp-server primary ipv4 192.168.107.165
 auto-loopback vrf default
  interface Loopback1
   primary-address 1.1.1.1
  !
 !
 cp-association retry-count 10
 <mark>l2tp enable</mark>
 secondary-address-update enable
!
</code>
</pre>
</div>

**Note**: NAL stands for Network Adaptation Layer for Cloud Native BNG in IOS-XR
{: .notice--info}
**Note**: cnBNG CP and UP doesnot require to be on same LAN, they need L3 connectivity for peering
{: .notice--info}

We need to create a Loopback for cnBNG internal use on ASR9k.

```
interface Loopback1
 ipv6 enable
```


### DHCP Configuration
This is where we associate access interfaces with cnBNG DHCP profile. cnBNG specific DHCP profile makes sure DHCP packets are punted to cnBNG CP through CPRi/GTP-u tunnel. Since PPPoE PTA subscribers use IPCP for IPv4 address assignment, dhcp ipv4 profile is not needed for PPPoE PTA subscribers.
```
dhcp ipv6
 profile cnbng_v6 cnbng
 !
 interface Bundle-Ether24 cnbng profile cnbng6
 interface Bundle-Ether25 cnbng profile cnbng6
```

### Access Interface Configuration
We define and associate access interface to cnBNG. This way control packets (based on configurations) get routed to the cnBNG CP. The contruct follows ASR9k Integarted BNG model, if you are familiar with.

```
interface Bundle-Ether24
 mtu 9216
 ipv4 address 172.24.0.2 255.255.255.0
 ipv6 address 2001:172:24::2/64
 load-interval 30
 lns enable
!
interface Bundle-Ether25
 mtu 9216
 ipv4 address 172.25.0.2 255.255.255.0
 ipv6 address 2001:172:25::2/64
 load-interval 30
 lns enable
!
```

## Radius Profile
Following are Freeradius profiles used in this tutorial. Profile-1 is for PPPoE PTA session and Profile-2 is for PPPoE LAC session.

**Profile-1**: Dynamic IP Assignment
```
lns-dynamic Cleartext-Password:="cisco"
  cisco-avpair += "subscriber:inacl=myACL",
  cisco-avpair += "subscriber:ipv6_inacl=myACL",
  cisco-avpair += "subscriber:ipv6_outacl=myACL",
  cisco-avpair += "subscriber:outacl=myACL",
  cisco-avpair += "subscriber:sub-qos-policy-in=PM_Plan_100mbps_input",
  cisco-avpair += "subscriber:sub-qos-policy-out=PM_Plan_100mbps_output",
  Cisco-AVPair += "strict-rpf=1",
  Cisco-AVPair += "ipv6-strict-rpf=1",
  Framed-route += "214.5.0.0/22",
  Framed-IPv6-route += "2001:214:5::/64"
```

**Profile-2**: Static IP Assignment including Framed-route and Framed-IPv6-route
```
lns-static Cleartext-Password:="cisco"
  cisco-avpair += "subscriber:inacl=myACL",
  cisco-avpair += "subscriber:ipv6_inacl=myACL",
  cisco-avpair += "subscriber:ipv6_outacl=myACL",
  cisco-avpair += "subscriber:outacl=myACL",
  cisco-avpair += "subscriber:sub-qos-policy-in=PM_Plan_100mbps_input",
  cisco-avpair += "subscriber:sub-qos-policy-out=PM_Plan_100mbps_output",
  Cisco-AVPair += "strict-rpf=1",
  Cisco-AVPair += "ipv6-strict-rpf=1",
  Framed-IP-Address += "112.0.0.3",
  Cisco-AVPair += "addrv6=2001:112::3",
  Delegated-IPv6-Prefix += "2001:111:0:3::/64",
  Framed-route += "214.6.0.0/22",
  Framed-IPv6-route += "2001:214:6::/64" 
```

## Verifications

- Verify subscriber sessions are up on cnBNG CP ops-center

<div class="highlighter-rouge">
<pre class="highlight">
<code>
[cnbng-tme-lab-2024/bng] bng# show subscriber session detail 
Fri Dec  6  09:39:17.753 UTC+00:00
subscriber-details 
{
  "subResponses": [
    {
      "subLabel": "16777218",
      "acct-sess-id": "cnbng-tme-lab-2024_DC_16777218",
      "upf": "ASR9k-1",
      "port-id": "Bundle-Ether24",
      "up-subs-id": "2147499664",
      "sesstype": "lns",
      "state": "established",
      "subCreateTime": "Fri, 06 Dec 2024 09:17:29 UTC",
      "dhcpAuditId": 1,
      "pppAuditId": 4,
      "transId": "4",
      "subsAttr": {
        "attrs": {
          "Authentic": "RADIUS(1)",
          "Framed-Protocol": "PPP(1)",
          "Interface-Id": "0x6c062813ada1085f",
          <mark>"addr": "112.0.0.3",</mark>
          <mark>"addrv6": "2001:112::3",</mark>
          "authen-type": "chap(2)",
          "challenge": "0xe9bdf835f6df262ed41d8cd98f00b204",
          "connect-progress": "DUAL_STACK_OPEN(249)",
          <mark>"delegated-prefix": "2001:111:0:3::/64",</mark>
          "dhcpv6-client-id": "0x000100016752bc27001094000015",
          "id": "1",
          "protocol-type": "ppp(2)",
          "response": "0xd7ad3deab0ab1ecaeaf2d3efe0af2199",
          "service-type": "Framed(2)",
          "string-session-id": "cnbng-tme-lab-2024_DC_16777218",
          <mark>"tunnel-client-auth-id": "lns.cisco.com",</mark>
          <mark>"tunnel-client-endpoint": "200.0.0.3",</mark>
          "tunnel-connection-id": "951517185",
          "tunnel-medium-type": "IPv4(1)",
          "tunnel-server-auth-id": "lns.cisco.com",
          <mark>"tunnel-server-endpoint": "172.0.0.2",</mark>
          "tunnel-type": "L2TP(1)",
          <mark>"username": "lns-static",</mark>
          "vrf": "default"
        }
      },
      "subcfgInfo": {
        "committedAttrs": {
          "attrs": {
            "accounting-list": "aaa_pppoe",
            "acct-interval": "60",
            "addr": "112.0.0.3",
            "addr-pool": "pool-Silver",
            "addrv6": "2001:112::3",
            "delegated-prefix": "2001:111:0:3::/64",
            "inacl": "myACL",
            "ipv4-mtu": "1500",
            "ipv6-route": "2001:214:6::/64",
            "ipv6-strict-rpf": "true",
            "ipv6_inacl": "myACL",
            "ipv6_outacl": "myACL",
            "outacl": "myACL",
            "ppp-authentication": "chap,pap",
            "ppp-ipcp-reneg-ignore": "true",
            "ppp-ipv6cp-reneg-ignore": "true",
            "ppp-keepalive-disable": "true",
            "ppp-lcp-reneg-ignore": "true",
            "ppp-max-bad-auth": "4",
            "ppp-max-failure": "5",
            "ppp-timeout-abs-minutes": "1440",
            "ppp-timeout-authentication": "5",
            "ppp-timeout-retry": "4",
            "route": "214.6.0.0/22",
            "session-acct-enabled": "true",
            <mark>"strict-rpf": "true",</mark>
            <mark>"sub-qos-policy-in": "PM_Plan_100mbps_input",</mark>
            <mark>"sub-qos-policy-out": "PM_Plan_100mbps_output",</mark>
            "vrf": "default"
          }
        },
        "activatedServices": [
          {
            "serviceName": "ft-pppoe-lns-1",
            "serviceAttrs": {
              "attrs": {}
            }
          }
        ]
      },
      "smupstate": "smUpSessionCreated",
      "v4AfiState": "Up",
      "v6AfiState": "Up",
      "interimInterval": "60",
      "interimSentToUp": "60",
      "sessionAccounting": "enable",
      "serviceAccounting": "disable",
      "upAttr": {
        "attrs": {
          "Interface-Id": "0x6c062813ada1085f",
          "addr": "112.0.0.3",
          "addrv6": "2001:112::3",
          "delegated-prefix": "2001:111:0:3::/64",
          "l2tp-df-reflect": "true",
          "ppp-local-magic-number": "2766053726",
          "ppp-mtu": "1500",
          "ppp-peer-magic-number": "580247585",
          "tunnel-tos-reflect": "true"
        }
      },
      <mark>"v4FramedRoute": [</mark>
        "214.6.0.0/22"
      ],
      <mark>"v6FramedRoute": [</mark>
        "2001:214:6::/64"
      ],
      "chargingInfo": {
        "sessionType": "CHARGING_AF_IPv4_IPv6",
        "sessionAccounting": {
          "periodicInterval": 60,
          "accountingProvision": "Enable",
          "stateInfo": "CHARGING_STATE_START_ACK",
          "accountingStart": {
            "reqStartSuccess": "Fri, 06 Dec 2024 09:17:33 UTC"
          },
          "accountingUpdate": {
            "reqInterimSuccess": "Fri, 06 Dec 2024 09:38:33 UTC",
            "periodicAccountingProvision": "Enable",
            "InterimIntervalTimeout": 60,
            "totalInterimReq": 23,
            "totalInterimFailure": 0
          },
          "accountingStop": {},
          "sessionDataStats": {
            "inputPkts": 2191894,
            "outputPkts": 2191879,
            "inputOctet": 429607952,
            "outputOctet": 425224360
          },
          "acct-sess-id": "cnbng-tme-lab-2024_DC_16777218",
          "UPFDataStats": {
            "ASR9k-1": {
              "inputPkts": 2191894,
              "outputPkts": 2191879,
              "inputOctet": 429607952,
              "outputOctet": 425224360
            }
          }
        }
      },
      "sess-events": [
        "Time, Event, Status",
        "2024-12-06 09:17:29.270320169 +0000 UTC, SessionCreate, success",
        "2024-12-06 09:17:33.307749887 +0000 UTC, SessionActivate, success",
        "2024-12-06 09:17:33.630390434 +0000 UTC, N4-Create:ASR9k-1, PASS",
        "2024-12-06 09:17:33.631082571 +0000 UTC, SessionUpdate, success",
        "2024-12-06 09:17:33.67049301 +0000 UTC, SessionUpdate, success",
        "2024-12-06 09:35:33.303059217 +0000 UTC, SessionUpdate, success"
      ]
    },
    {
      "subLabel": "16777217",
      "acct-sess-id": "cnbng-tme-lab-2024_DC_16777217",
      "upf": "ASR9k-1",
      "port-id": "Bundle-Ether24",
      "up-subs-id": "2147499648",
      "sesstype": "lns",
      "state": "established",
      "subCreateTime": "Fri, 06 Dec 2024 09:17:23 UTC",
      "dhcpAuditId": 1,
      "pppAuditId": 4,
      "transId": "4",
      "subsAttr": {
        "attrs": {
          "Authentic": "RADIUS(1)",
          "Framed-Protocol": "PPP(1)",
          "Interface-Id": "0x7940d8754c682e87",
          <mark>"addr": "192.168.4.2",</mark>
          <mark>"addrv6": "2001:192:168::1000",</mark>
          "authen-type": "chap(2)",
          "challenge": "0x6ff2b3842904db56d41d8cd98f00b204",
          "connect-progress": "DUAL_STACK_OPEN(249)",
          "delegated-prefix": "2008::/64",
          "dhcpv6-client-id": "0x000100016752bc21001094000014",
          "id": "1",
          "protocol-type": "ppp(2)",
          "response": "0xdf494a18e4b6514b84b57b0029922da0",
          "service-type": "Framed(2)",
          "string-session-id": "cnbng-tme-lab-2024_DC_16777217",
          <mark>"tunnel-client-auth-id": "lns.cisco.com",</mark>
          <mark>"tunnel-client-endpoint": "200.0.0.2",</mark>
          "tunnel-connection-id": "153944065",
          "tunnel-medium-type": "IPv4(1)",
          "tunnel-server-auth-id": "lns.cisco.com",
          <mark>"tunnel-server-endpoint": "172.0.0.2",</mark>
          "tunnel-type": "L2TP(1)",
          <mark>"username": "lns-dynamic",</mark>
          "vrf": "default"
        }
      },
      "subcfgInfo": {
        "committedAttrs": {
          "attrs": {
            "accounting-list": "aaa_pppoe",
            "acct-interval": "60",
            "addr-pool": "pool-Silver",
            "inacl": "myACL",
            "ipv4-mtu": "1500",
            "ipv6-route": "2001:214:5::/64",
            "ipv6-strict-rpf": "true",
            "ipv6_inacl": "myACL",
            "ipv6_outacl": "myACL",
            "outacl": "myACL",
            "ppp-authentication": "chap,pap",
            "ppp-ipcp-reneg-ignore": "true",
            "ppp-ipv6cp-reneg-ignore": "true",
            "ppp-keepalive-disable": "true",
            "ppp-lcp-reneg-ignore": "true",
            "ppp-max-bad-auth": "4",
            "ppp-max-failure": "5",
            "ppp-timeout-abs-minutes": "1440",
            "ppp-timeout-authentication": "5",
            "ppp-timeout-retry": "4",
            "route": "214.5.0.0/22",
            "session-acct-enabled": "true",
            <mark>"strict-rpf": "true",</mark>
            <mark>"sub-qos-policy-in": "PM_Plan_100mbps_input",</mark>
            <mark>"sub-qos-policy-out": "PM_Plan_100mbps_output",</mark>
            "vrf": "default"
          }
        },
        "activatedServices": [
          {
            "serviceName": "ft-pppoe-lns-1",
            "serviceAttrs": {
              "attrs": {}
            }
          }
        ]
      },
      "smupstate": "smUpSessionCreated",
      "v4AfiState": "Up",
      "v6AfiState": "Up",
      "interimInterval": "60",
      "interimSentToUp": "60",
      "sessionAccounting": "enable",
      "serviceAccounting": "disable",
      "upAttr": {
        "attrs": {
          "Interface-Id": "0x7940d8754c682e87",
          "addr": "192.168.4.2",
          "addrv6": "2001:192:168::1000",
          "delegated-prefix": "2008::/64",
          "l2tp-df-reflect": "true",
          "ppp-local-magic-number": "3688316752",
          "ppp-mtu": "1500",
          "ppp-peer-magic-number": "1813240529",
          "tunnel-tos-reflect": "true"
        }
      },
      <mark>"v4FramedRoute": [</mark>
        "214.5.0.0/22"
      ],
      <mark>"v6FramedRoute": [</mark>
        "2001:214:5::/64"
      ],
      "chargingInfo": {
        "sessionType": "CHARGING_AF_IPv4_IPv6",
        "sessionAccounting": {
          "periodicInterval": 60,
          "accountingProvision": "Enable",
          "stateInfo": "CHARGING_STATE_START_ACK",
          "accountingStart": {
            "reqStartSuccess": "Fri, 06 Dec 2024 09:17:28 UTC"
          },
          "accountingUpdate": {
            "reqInterimSuccess": "Fri, 06 Dec 2024 09:38:28 UTC",
            "periodicAccountingProvision": "Enable",
            "InterimIntervalTimeout": 60,
            "totalInterimReq": 23,
            "totalInterimFailure": 0
          },
          "accountingStop": {},
          "sessionDataStats": {
            "inputPkts": 2191877,
            "outputPkts": 2191863,
            "inputOctet": 429604622,
            "outputOctet": 425221328
          },
          "acct-sess-id": "cnbng-tme-lab-2024_DC_16777217",
          "UPFDataStats": {
            "ASR9k-1": {
              "inputPkts": 2191877,
              "outputPkts": 2191863,
              "inputOctet": 429604622,
              "outputOctet": 425221328
            }
          }
        }
      },
      "sess-events": [
        "Time, Event, Status",
        "2024-12-06 09:17:23.585201637 +0000 UTC, SessionCreate, success",
        "2024-12-06 09:17:27.623952571 +0000 UTC, SessionActivate, success",
        "2024-12-06 09:17:28.05379705 +0000 UTC, N4-Create:ASR9k-1, PASS",
        "2024-12-06 09:17:28.054551382 +0000 UTC, SessionUpdate, success",
        "2024-12-06 09:17:28.136872388 +0000 UTC, SessionUpdate, success",
        "2024-12-06 09:35:27.224115757 +0000 UTC, SessionUpdate, success"
      ]
    }
  ]
}
</code>
</pre>
</div>

- Verify l2tp tunnels on cnBNG CP ops-center.

```
[cnbng-tme-lab-2024/bng] bng# show l2tp-tunnel detail 
Fri Dec  6  09:39:30.338 UTC+00:00
tunnel-details 
{
  "tunResponses": [
    {
      "state": "established",
      "profileName": "lns-up1",
      "tunnelType": "lns",
      "sessionCount": 1,
      "IDs Allocated": 1,
      "routerID": "ASR9k-1",
      "srcIP": "172.0.0.2",
      "dstIP": "200.0.0.2",
      "localTunnelID": 2349,
      "remoteTunnelID": 1,
      "tunnelClientAuthID": "lns.cisco.com",
      "tunnelServerAuthID": "lns.cisco.com"
    },
    {
      "state": "established",
      "profileName": "lns-up1",
      "tunnelType": "lns",
      "sessionCount": 1,
      "IDs Allocated": 1,
      "routerID": "ASR9k-1",
      "srcIP": "172.0.0.2",
      "dstIP": "200.0.0.3",
      "localTunnelID": 14519,
      "remoteTunnelID": 1,
      "tunnelClientAuthID": "lns.cisco.com",
      "tunnelServerAuthID": "lns.cisco.com"
    }
  ]
}
```

- Verify that the subscriber session is up and working on cnBNG UP

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RSP0/CPU0:ASR9k-1#show cnbng-nal subscriber all

Fri Dec  6 09:30:46.924 UTC

Location: 0/RSP0/CPU0
Codes: CN - Connecting, CD - Connected, AC - Activated,
       ID - Idle, DN - Disconnecting, IN - Initializing
       UN - Unknown


CPID(hex)  Interface               State  Mac Address     Subscriber IP Addr / Prefix (Vrf) Ifhandle
---------------------------------------------------------------------------------------------------
1000002    BE24.lns2147499664      AC     0000.0000.0000  112.0.0.3 (default) 0x10260   
                                                          214.6.0.0/22 (Framed IPv4)
1000001    BE24.lns2147499648      AC     0000.0000.0000  192.168.4.2 (default) 0x10220   
                                                          214.5.0.0/22 (Framed IPv4)
Session-count: 2

RP/0/RSP0/CPU0:ASR9k-1#show subscriber running-config interface name BE24.lns2147499664

Fri Dec  6 09:32:27.710 UTC
Building configuration...
!! IOS XR Configuration 24.4.1.07I
dynamic-template
 type user-profile U00003e90
  ipv4 access-group myACL ingress
  ipv4 access-group myACL egress
  ipv4 mtu 1500
  ipv4 unnumbered Loopback1
  service-policy input PM_Plan_100mbps_input
  service-policy output PM_Plan_100mbps_output
  ipv6 access-group myACL ingress
  ipv6 access-group myACL egress
  ipv6 enable
 !
!
end

* Suffix indicates the configuration item can be added by aaa server only
RP/0/RSP0/CPU0:ASR9k-1#show policy-map applied interface             BE24.lns2147499664

Fri Dec  6 09:32:27.973 UTC

Input policy-map applied to Bundle-Ether24.lns2147499664:

  policy-map PM_Plan_100mbps_input
   class class-default
    police rate 100 mbps 
     exceed-action drop
    ! 
   ! 

Output policy-map applied to Bundle-Ether24.lns2147499664:

  policy-map PM_Plan_100mbps_output
   class class-default
    service-policy PM_Plan_100mbps_Child
    shape average 100 mbps 3 mbytes 
   ! 

Child policy-map(s) of policy-map PM_Plan_100mbps_output: 

  policy-map PM_Plan_100mbps_Child
   class DSCP-EF
    police rate 1 mbps burst 200 bytes 
    ! 
    priority level 1 
   ! 
   class CS3
    shape average 5 mbps 1 mbytes 
   ! 
   class class-default
   ! 
   end-policy-map
  ! 
RP/0/RSP0/CPU0:ASR9k-1#show subscriber running-config interface name BE24.lns2147499648

Fri Dec  6 09:32:30.559 UTC
Building configuration...
!! IOS XR Configuration 24.4.1.07I
dynamic-template
 type user-profile U00003e80
  ipv4 access-group myACL ingress
  ipv4 access-group myACL egress
  ipv4 mtu 1500
  ipv4 unnumbered Loopback1
  service-policy input PM_Plan_100mbps_input
  service-policy output PM_Plan_100mbps_output
  ipv6 access-group myACL ingress
  ipv6 access-group myACL egress
  ipv6 enable
 !
!
end

* Suffix indicates the configuration item can be added by aaa server only
RP/0/RSP0/CPU0:ASR9k-1#show policy-map applied interface             BE24.lns2147499648

Fri Dec  6 09:32:30.898 UTC

Input policy-map applied to Bundle-Ether24.lns2147499648:

  policy-map PM_Plan_100mbps_input
   class class-default
    police rate 100 mbps 
     exceed-action drop
    ! 
   ! 

Output policy-map applied to Bundle-Ether24.lns2147499648:

  policy-map PM_Plan_100mbps_output
   class class-default
    service-policy PM_Plan_100mbps_Child
    shape average 100 mbps 3 mbytes 
   ! 

Child policy-map(s) of policy-map PM_Plan_100mbps_output: 

  policy-map PM_Plan_100mbps_Child
   class DSCP-EF
    police rate 1 mbps burst 200 bytes 
    ! 
    priority level 1 
   ! 
   class CS3
    shape average 5 mbps 1 mbytes 
   ! 
   class class-default
   ! 
   end-policy-map
  ! 
</code>
</pre>
</div>

**Note**: The ACL and QoS policies applied on subscriber interface must be defined on ASR9k (cnBNG UP), prior to subscriber session bring-up.
{: .notice--info}


