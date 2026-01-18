# Install OAI-CN5G-UPF(eBPF/XDP UPF) on Host

This briefly describes the steps and configuration to build and install [OAI-CN5G-UPF](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf).
**It is intended to be prepared for use with [Open5GS](https://github.com/open5gs/open5gs) and [free5GC](https://github.com/free5gc/free5gc).**

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Simple Overview of OAI-CN5G-UPF and Data Network Gateway](#overview)
- [Build OAI-CN5G-UPF on VM-UP](#build)
  - [Confirmed Version List](#ver_list)
  - [Install required packages](#install_pkg)
  - [Get patches](#get_patch)
  - [Clone OAI-CN5G-UPF](#clone)
  - [Build and Install OAI-CN5G-UPF](#build_install)
- [Setup OAI-CN5G-UPF on VM-UP](#setup_up)
  - [Create configuration file](#conf)
  - [Note for smf.yaml of Open5GS](#open5gs)
- [Run OAI-CN5G-UPF on VM-UP](#run)
- [Setup Data Network Gateway on VM-DN](#setup_dn)
- [How to capture packets on DPDK ports](#pcap)
- [Sample Configurations](#sample_conf)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Simple Overview of OAI-CN5G-UPF and Data Network Gateway

This describes a simple configuration of OAI-CN5G-UPF and Data Network Gateway, focusing on U-Plane.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=800px></img>

The eBPF/XDP UPF used is as follows.
- eBPF/XDP UPF - OAI-CN5G-UPF v2.2.0 (2025.12.13) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM-UP | OAI-CN5G-UPF U-Plane | 192.168.0.151/24 | Ubuntu 22.04 | 1 | 6GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Model | Linux Bridge | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- | --- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | N3 | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | N4 | -- |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | N6 | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | N6 | -- |

Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | N3 |
| vmbr4 | 192.168.14.0/24 | N4 |
| vmbr6 | 192.168.16.0/24 | N6 |

The Network Instance, DNN and S-NSSAI are configured as follows.
| Network Instance | DNN | SST | SD |
| --- | --- | --- | --- |
| internet | internet | 1 | -- |

<a id="build"></a>

## Build and Install OAI-CN5G-UPF on VM-UP

<a id="ver_list"></a>

### Confirmed Version List

I simply confirmed the operation of the following versions.

| Version | Commit | Date |
| --- | --- | --- |
| 2.2.0 | e025cdfb3a9c18a228f2efe36bd06b9de998554c | 2025.12.13 |

<a id="install_pkg"></a>

### Install required packages

```
# apt install arping
```

<a id="get_patch"></a>

### Get patches

First, get the patches for the following merge requests of OAI-CN5G-UPF to work with free5GC.

- [fix: PFCP Session Establishment Failures with Free5GC SMF](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf/-/merge_requests/88)
  ```
  # wget https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf/-/merge_requests/88.diff -O 88.patch
  ```
- [fix: Resolve UL/DL traffic failure in non-host-network mode via eBPF FIB lookup](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf/-/merge_requests/85)
  ```
  # wget https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf/-/merge_requests/85.diff -O 85.patch
  ```
- [measurement-ie.patch](https://gitlab.eurecom.fr/-/project/5331/uploads/a477009219a1535fa1f4ab85aaba422a/measurement-ie.patch) (linked [here](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf/-/merge_requests/88))
  ```
  # wget https://gitlab.eurecom.fr/-/project/5331/uploads/a477009219a1535fa1f4ab85aaba422a/measurement-ie.patch
  ```
And, I have created a [patch](https://github.com/s5uishida/install_oai_upf/blob/main/patches/common-src_3gpp_interface_type.patch) that enables the following IEs to handle `3GPP Interface Type IE` for working with Open5GS., so get it.
```
3GPP TS 29.244 Table 7.5.2.2-2: PDI IE within PFCP Session Establishment Request
3GPP TS 29.244 Table 7.5.4.3-2: Update Forwarding Parameters IE in the Update FAR IE
3GPP TS 29.244 Table 7.5.2.3-2: Forwarding Parameters IE in FAR
```
```
# wget https://github.com/s5uishida/install_oai_upf/raw/refs/heads/main/patches/common-src_3gpp_interface_type.patch
```

<a id="clone"></a>

### Clone OAI-CN5G-UPF

Download and change to tag `v2.2.0`.
```
# cd ~
# git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf.git
# cd oai-cn5g-upf
# git checkout refs/tags/v2.2.0
```

<a id="build_install"></a>

### Build and Install OAI-CN5G-UPF

First, check installed software necessary to build and run UPF.
```
# cd ~/oai-cn5g-upf/build/scripts
# ./build_upf -I -f
```
And, apply the patches downloaded.
```
# cd ~/oai-cn5g-upf
# patch -p1 < ~/85.patch
# patch -p1 < ~/88.patch
# cd src/common-src
# patch -p1 < ~/measurement-ie.patch
# patch -p1 < ~/common-src_3gpp_interface_type.patch
```
Then, build and Install OAI-CN5G-UPF
```
# cd ~/oai-cn5g-upf/build/scripts
# ./build_upf -c -V -b Release -j
# cp ~/oai-cn5g-upf/etc/config.yaml /usr/local/etc/oai/config.yaml.orig
# echo "/usr/lib64" >> /etc/ld.so.conf.d/libbpf.conf
# ldconfig
```
Finally, prepare the directory `openair-upf` to run UPF and the `tc` module to control QER.
```
# mkdir -p ~/openair-upf/bin
# cp ~/oai-cn5g-upf/build/upf/build/upf_app/bpf/CMakeFiles/qer_tc.dir/rules/qer/qer_tc_kernel.c.o ~/openair-upf/bin/
# ln -s /root/openair-upf /openair-upf
```

<a id="setup_up"></a>

## Setup OAI-CN5G-UPF on VM-UP

First, down `ens18` which is the default gateway interface of VM-UP.
```
# ip link set dev ens18 down
```

<a id="conf"></a>

### Create configuration file

First, create a configuration file `config.yaml` in the directory `openair-upf` as follows.
Please refer `/usr/local/etc/oai/config.yaml.orig` for the configuration.

`openair-upf/config.yaml`
```yaml
log_level:
  general: info

register_nf:
  general: no

http_version: 2

nfs:
  nrf:
    host: 192.168.14.111
    sbi:
      port: 8080
      api_version: v1
      interface_name: ens21
  smf:
    host: 192.168.14.111
    sbi:
      port: 8080
      api_version: v1
      interface_name: ens21
    n4:
      interface_name: ens21
      port: 8805
  upf:
    host: 192.168.14.151
    sbi:
      port: 8080
      api_version: v1
      interface_name: ens21
    n3:
      interface_name: ens20
      port: 2152
      nwi: "internet"
    n4:
      interface_name: ens21
      port: 8805
    n6:
      interface_name: ens22
      nwi: "internet"
    n9:
      interface_name: ens20
      port: 2152

snssais:
  - &slice1
    sst: 1

upf:
  support_features:
    enable_bpf_datapath: yes
    enable_qos: yes
    max_upf_interfaces: 3
    max_upf_redirect_interfaces: 2
    max_pdu_session: 10000 
    max_pdrs_per_pdu_session: 8
    max_qos_flows_per_pdu_session: 8
    max_sdf_filters_per_pdu_session: 8
    max_arp_entries: 2
  remote_n6_gw: 192.168.16.152
  upf_info:
    sNssaiUpfInfoList:
      - sNssai: *slice1
        dnnUpfInfoList:
          - dnn: "internet"

dnns:
  - dnn: "internet"
    pdu_session_type: "IPV4"
    ipv4_subnet: "10.45.0.0/16"
```
Of these, the following settings will not be used, but they are set for the purpose of running UPF.
```yaml
nfs:
  nrf:
    host: 192.168.14.111
    sbi:
      port: 8080
      api_version: v1
      interface_name: ens21
  smf:
    host: 192.168.14.111
    sbi:
      port: 8080
      api_version: v1
      interface_name: ens21
    n4:
      interface_name: ens21
      port: 8805
  upf:
    ...
    sbi:
      port: 8080
      api_version: v1
      interface_name: ens21
```

<a id="open5gs"></a>

### Note for smf.yaml of Open5GS

When working with Open5GS SMF, add the following settings to `smf.yaml`.

`smf.yaml`
```
global:
...
  parameter:
    use_upg_vpp: true
```

<a id="run"></a>

## Run OAI-CN5G-UPF on VM-UP

```
# cd ~/openair-upf
# upf -c config.yaml -o
Trying to read .yaml configuration file: config.yaml
LTTNG Tracing disabled at build-time!
[2026-01-18 08:53:43.481] [upf_app] [start] Options parsed
[2026-01-18 08:53:43.481] [config ] [info] Reading NF configuration from config.yaml
[2026-01-18 08:53:43.491] [config ] [debug] Validating configuration of log_level
[2026-01-18 08:53:43.496] [config ] [info] ==== OPENAIRINTERFACE upf vBranch: HEAD Abrev. Hash: e025cdf Date: Sat Dec 13 00:09:59 2025 +0100 ====
[2026-01-18 08:53:43.496] [config ] [info] Basic Configuration:
[2026-01-18 08:53:43.496] [config ] [info]   - log_level..................................: info
[2026-01-18 08:53:43.496] [config ] [info]   - register_nf................................: No
[2026-01-18 08:53:43.496] [config ] [info]   - http_version...............................: 2
[2026-01-18 08:53:43.496] [config ] [info]   TLS:
[2026-01-18 08:53:43.496] [config ] [info]     - Enable TLS.................................: No
[2026-01-18 08:53:43.496] [config ] [info]   - HTTP Request Timeout.......................: 3000 (ms)
[2026-01-18 08:53:43.496] [config ] [info] UPF Configuration:
[2026-01-18 08:53:43.496] [config ] [info]   - host.......................................: 192.168.14.151
[2026-01-18 08:53:43.496] [config ] [info]   - SBI
[2026-01-18 08:53:43.496] [config ] [info]     + URL......................................: 192.168.14.151:8080
[2026-01-18 08:53:43.496] [config ] [info]     + API Version..............................: v1
[2026-01-18 08:53:43.496] [config ] [info]     + IPv4 Address ............................: 192.168.14.151
[2026-01-18 08:53:43.496] [config ] [info]   - N3:
[2026-01-18 08:53:43.496] [config ] [info]     + Port.....................................: 2152
[2026-01-18 08:53:43.496] [config ] [info]     + IPv4 Address ............................: 192.168.13.151
[2026-01-18 08:53:43.496] [config ] [info]     + MTU......................................: 1500
[2026-01-18 08:53:43.496] [config ] [info]     + Interface name: .........................: ens20
[2026-01-18 08:53:43.496] [config ] [info]     + Network Instance.........................: internet
[2026-01-18 08:53:43.496] [config ] [info]   - N4:
[2026-01-18 08:53:43.496] [config ] [info]     + Port.....................................: 8805
[2026-01-18 08:53:43.496] [config ] [info]     + IPv4 Address ............................: 192.168.14.151
[2026-01-18 08:53:43.496] [config ] [info]     + MTU......................................: 1500
[2026-01-18 08:53:43.496] [config ] [info]     + Interface name: .........................: ens21
[2026-01-18 08:53:43.496] [config ] [info]   - N6:
[2026-01-18 08:53:43.496] [config ] [info]     + Port.....................................: 2152
[2026-01-18 08:53:43.496] [config ] [info]     + IPv4 Address ............................: 192.168.16.151
[2026-01-18 08:53:43.496] [config ] [info]     + MTU......................................: 1500
[2026-01-18 08:53:43.496] [config ] [info]     + Interface name: .........................: ens22
[2026-01-18 08:53:43.496] [config ] [info]     + Network Instance.........................: internet
[2026-01-18 08:53:43.496] [config ] [info]   - Instance ID................................: 0
[2026-01-18 08:53:43.496] [config ] [info]   - Remote N6 Gateway..........................: 192.168.16.152
[2026-01-18 08:53:43.496] [config ] [info]   - Support Features:
[2026-01-18 08:53:43.496] [config ] [info]     + Enable BPF Datapath......................: Yes
[2026-01-18 08:53:43.496] [config ] [info]     + Enable QoS...............................: Yes
[2026-01-18 08:53:43.496] [config ] [info]     + Max upf interfaces.......................: 3
[2026-01-18 08:53:43.496] [config ] [info]     + Max upf redirect interfaces..............: 2
[2026-01-18 08:53:43.496] [config ] [info]     + Max PDU Session..........................: 10000
[2026-01-18 08:53:43.496] [config ] [info]     + Max PDRs Per PDU Session.................: 8
[2026-01-18 08:53:43.496] [config ] [info]     + Max Qos Flows Per PDU Session............: 8
[2026-01-18 08:53:43.496] [config ] [info]     + Max SDF Filters Per PDU Session..........: 8
[2026-01-18 08:53:43.496] [config ] [info]     + Max ARP Entries..........................: 2
[2026-01-18 08:53:43.496] [config ] [info]     + Enable SNAT..............................: No
[2026-01-18 08:53:43.496] [config ] [info]     + enable_fr................................: No
[2026-01-18 08:53:43.496] [config ] [info]     + Enable Ethernet PDU Session..............: No
[2026-01-18 08:53:43.496] [config ] [info]     + Ignore QFI For Uplink Classification.....: Yes
[2026-01-18 08:53:43.496] [config ] [info]   + upf_info:
[2026-01-18 08:53:43.496] [config ] [info]     - snssai_upf_info_item:
[2026-01-18 08:53:43.496] [config ] [info]       + snssai:
[2026-01-18 08:53:43.496] [config ] [info]         - sst..................................: 1
[2026-01-18 08:53:43.496] [config ] [info]         - sd...................................: FFFFFF
[2026-01-18 08:53:43.496] [config ] [info]       + dnns:
[2026-01-18 08:53:43.496] [config ] [info]         - dnn..................................: internet
[2026-01-18 08:53:43.496] [config ] [info] Peer NF Configuration:
[2026-01-18 08:53:43.496] [config ] [info]   NRF:
[2026-01-18 08:53:43.496] [config ] [info]     - host.....................................: 192.168.14.111
[2026-01-18 08:53:43.496] [config ] [info]     - SBI
[2026-01-18 08:53:43.496] [config ] [info]       + URL....................................: 192.168.14.111:8080
[2026-01-18 08:53:43.496] [config ] [info]       + API Version............................: v1
[2026-01-18 08:53:43.496] [config ] [info]   SMF:
[2026-01-18 08:53:43.496] [config ] [info]     - host.....................................: 192.168.14.111
[2026-01-18 08:53:43.496] [config ] [info]     - SBI
[2026-01-18 08:53:43.496] [config ] [info]       + URL....................................: 192.168.14.111:8080
[2026-01-18 08:53:43.496] [config ] [info]       + API Version............................: v1
[2026-01-18 08:53:43.496] [config ] [info] DNNs:
[2026-01-18 08:53:43.496] [config ] [info] - DNN:
[2026-01-18 08:53:43.496] [config ] [info]     + DNN......................................: internet
[2026-01-18 08:53:43.496] [config ] [info]     + PDU session type.........................: IPV4
[2026-01-18 08:53:43.496] [config ] [info]     + IPv4 subnet..............................: 10.45.0.0/16
[2026-01-18 08:53:43.496] [config ] [info]     + DNS Settings:
[2026-01-18 08:53:43.496] [config ] [info]       - primary_dns_ipv4.......................: 8.8.8.8
[2026-01-18 08:53:43.496] [config ] [info]       - secondary_dns_ipv4.....................: 1.1.1.1
[2026-01-18 08:53:43.496] [upf_app] [info] HTTP Client successfully initiated on interface ens21 with timeout 3000 ms, HTTP version 2
[2026-01-18 08:53:43.496] [common] [start] Starting...
[2026-01-18 08:53:43.496] [common] [start] Started
[2026-01-18 08:53:43.496] [asc_cmd] [start] Starting...
[2026-01-18 08:53:43.496] [common] [info] Starting timer_manager_task
[2026-01-18 08:53:43.497] [asc_cmd] [start] Started
[2026-01-18 08:53:43.497] [upf_app] [start] Starting...
[2026-01-18 08:53:43.498] [pfcp   ] [info] pfcp_l4_stack created listening to 192.168.14.151:8805
[2026-01-18 08:53:43.499] [upf_n4 ] [start] Starting...
[2026-01-18 08:53:43.500] [upf_n4 ] [start] Started
[2026-01-18 08:53:43.500] [upf_app] [start] Started
[2026-01-18 08:53:43.500] [upf_app] [info] GTP interface: ens20
[2026-01-18 08:53:43.500] [upf_app] [info] Non-GTP interface: ens22
[2026-01-18 08:53:43.500] [upf_app] [info] Initializing PFCP Session Lookup BPF program...
[2026-01-18 08:53:43.500] [upf_app] [info] BPFProgram 1 is created!!!
[2026-01-18 08:53:43.502] [upf_app] [info] PDR Lookup Config: ignore_qfi_for_uplink=true
[2026-01-18 08:53:43.582] [upf_app] [info] Reference Point N3 Added to m_upf_interface Map
[2026-01-18 08:53:43.582] [upf_app] [info] Reference Point N6 Added to m_upf_interface Map
[2026-01-18 08:53:43.582] [upf_app] [info] Reference Point N4 Added to m_upf_interface Map
[2026-01-18 08:53:43.583] [upf_app] [info] BPF program xdp_handle_uplink hooked in ens20 XDP interface
[2026-01-18 08:53:43.583] [upf_app] [info] BPF program xdp_handle_shaping hooked in ens22 XDP interface
```
The link status of the network interfaces N3(ens20) and N6(ens22) is as follows.
```
# ip link show
...
4: ens20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether bc:24:11:cc:90:e0 brd ff:ff:ff:ff:ff:ff
    prog/xdp id 36 tag 6c60e72a1ce2d8a3 jited 
    altname enp0s20
...
6: ens22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether bc:24:11:0e:f3:90 brd ff:ff:ff:ff:ff:ff
    prog/xdp id 38 tag ac3f8821f6f1dc69 jited 
    altname enp0s22
...
```

<a id="setup_dn"></a>

## Setup Data Network Gateway on VM-DN

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, configure NAPT and routing to N6 IP address of OAI-CN5G-UPF.
```
# iptables -t nat -A POSTROUTING -s <DN> -j MASQUERADE
# ip route add <DN> via 192.168.16.151 dev ens20
```
**Note. Set `<DN>` according to the core network.  
ex) `10.45.0.0/16`**


<a id="pcap"></a>

## How to capture packets on XDP ports

There are two ways to do this.

1. [How to run `xdpdump`](https://github.com/xdp-project/xdp-tools/tree/main/xdp-dump)
2. [How to run `tcpdump` or `tshark` on another VM by configuring a bridge interface linked to a network interface for XDP](https://github.com/s5uishida/proxmox_ve_tips#enable_promisc)

---
With the above steps, OAI-CN5G-UPF(eBPF/XDP UPF) has been constructed.
You will be able to work OAI-CN5G-UPF with Open5GS and free5GC.
I would like to thank the excellent developers and all the contributors of OAI-CN5G-UPF.

<a id="sample_conf"></a>

## Sample Configurations

- [Open5GS 5GC & UERANSIM UE / RAN Sample Configuration - OAI-CN5G-UPF(eBPF/XDP UPF)](https://github.com/s5uishida/open5gs_5gc_ueransim_oai_upf_sample_config)
- [free5GC 5GC & UERANSIM UE / RAN Sample Configuration - OAI-CN5G-UPF(eBPF/XDP UPF)](https://github.com/s5uishida/free5gc_ueransim_oai_upf_sample_config)

<a id="changelog"></a>

## Changelog (summary)

- [2026.01.18] Initial release.
