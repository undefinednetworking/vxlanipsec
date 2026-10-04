# Open vSwitch (OVS) Network Security Lab: VXLAN, Geneve, IPsec, ML-KEM1024 & Certificates

This repository contains an automated multi-node VirtualBox environment provisioned via **Vagrant** to explore overlay networking, packet encapsulation, and tunnel encryption using **Open vSwitch (OVS)**, **VXLAN**, **Geneve**, and **IPsec (StrongSwan)**.

For detailed theory, architectural breakdowns, and step-by-step video walkthroughs, refer to the following resources:
* **Blog Article**: [Securing Networks: A Deep Dive into IPsec, Geneve, and Open vSwitch](https://medium.com/@undefinednetworking/securing-networks-a-deep-dive-into-ipsec-geneve-and-open-vswitch-665ab8b109c0)[cite: 1]
* **YouTube Video**: [IPsec, Geneve, and Open vSwitch Walkthrough](https://www.youtube.com/watch?v=_12egWUnd8I)[cite: 1]

---

## 📐 Architecture Overview

The lab provisions two Ubuntu 26.04 VMs (`node1` and `node2`) connected via a bridged public network. Each node runs Open vSwitch with two isolated internal interfaces routed across two distinct tunnel types:

1. **Cleartext Tunnel (VXLAN)**: Routes traffic between `int-clear` interfaces over UDP port 4789.
2. **Encrypted Tunnel (Geneve + IPsec)**: Routes traffic between `int-ipsec` interfaces using Geneve encapsulation over UDP port 6081, protected by OVS IPsec auto-tunneling (StrongSwan) with self signed Certificates.

```text
                     +---------------------------------------+
                     |         Bridged Host Network          |
                     |             192.168.0.0/24            |
                     +-------------------+-------------------+
                                         |
                     +-------------------+-------------------+
                     |                                       |
           +---------+---------+                   +---------+---------+
           |       node1       |                   |       node2       |
           | (192.168.0.201)   |                   | (192.168.0.202)   |
           |    [ Bridge br0 ] |                   |    [ Bridge br0 ] |
           |  int-clear        | -- VXLAN (4789) ->|  int-clear        |
           |  (10.100.0.1/24)  |   (Unencrypted)   |  (10.100.0.2/24)  |
           |                   |                   |                   |
           |  int-ipsec        | - Geneve + IPsec ->  int-ipsec        |
           |  (10.200.0.1/24)  | (6081 Encrypted)  |  (10.200.0.2/24)  |
           +-------------------+  (ML-KEM1024)     +-------------------+

```

---

## 🌐 IP Allocation & Port Mapping

### Node Interfaces

| Node | Public Network IP | Interface | Internal IP Subnet | Tunnel Type | Remote End |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **node1** | `192.168.0.201` | `int-clear`[cite: 1] | `10.100.0.1/24`[cite: 1] | VXLAN (VNI 100)[cite: 1] | `192.168.0.202`[cite: 1] |
| **node1** | `192.168.0.201`[cite: 1] | `int-ipsec`[cite: 1] | `10.200.0.1/24`[cite: 1] | Geneve + IPsec (VNI 200)[cite: 1] | `192.168.0.202`[cite: 1] |
| **node2** | `192.168.0.202`[cite: 1] | `int-clear`[cite: 1] | `10.100.0.2/24`[cite: 1] | VXLAN (VNI 100)[cite: 1] | `192.168.0.201`[cite: 1] |
| **node2** | `192.168.0.202`[cite: 1] | `int-ipsec`[cite: 1] | `10.200.0.2/24`[cite: 1] | Geneve + IPsec (VNI 200)[cite: 1] | `192.168.0.201`[cite: 1] |

### Firewall Rules & Ports Configured (UFW)

* **SSH**: `22/tcp`[cite: 1]
* **VXLAN Traffic**: `4789/udp`[cite: 1]
* **Geneve Traffic**: `6081/udp`[cite: 1]
* **IPsec Key Exchange (IKE / NAT-T)**: `500/udp`, `4500/udp`[cite: 1]
* **IPsec Payload Protocol**: `ESP`[cite: 1]

---

## 🚀 Quick Start

### Prerequisites

* [Vagrant](https://www.vagrantup.com/) (v2.2+)
* [VirtualBox](https://www.virtualbox.org/)
* Wireshark (installed on the host machine for packet analysis)[cite: 1]

### Deployment

1. **Clone the repository:**
   ```bash
   git clone https://github.com/undefinednetworking/vxlanipsec.git
   cd vxlanipsec ```

2. **Spin up both virtual machines:**

```
vagrant up
```
Note: Provisioning installs standard Open vSwitch, StrongSwan IPsec daemons, configures Netplan persistent internal interfaces, sets up custom OpenFlow rules, and configures a systemd service to persist flows on boot[cite: 1].

## 🧪 **Testing & Verification**
1. **Test Cleartext VXLAN Connectivity**
Log into node1 and ping node2 over the cleartext VXLAN network[cite: 1]:

```
vagrant ssh node1
ping -c 4 10.100.0.2
```

### 2. Test Encrypted Geneve + IPsec Connectivity

Ping `node2` over the IPsec-secured network[cite: 1]:
```bash
ping -c 4 10.200.0.2
```

### 3. Verify Open vSwitch Flows & Tunnels

Check bridge status and active OpenFlow rules on either node[cite: 1]:
```bash
# Check OVS configuration
sudo ovs-vsctl show

# Inspect active OpenFlow rules
sudo ovs-ofctl dump-flows br0
```

### 4. Packet Capture Analysis (Wireshark / tcpdump)

Promiscuous mode (`--nicpromisc2 allow-all`) is pre-enabled on VirtualBox interfaces to allow packet inspection directly on the host or inside the VMs[cite: 1]:

* **Unencrypted Capture**:
  Run `sudo tcpdump -i eth1 udp port 4789 -nn` on `node1` while pinging `10.100.0.2`. You will see unencrypted VXLAN packets containing the raw ICMP echo requests/replies[cite: 1].
* **Encrypted Capture**:
  Run `sudo tcpdump -i eth1 esp or udp port 500 or udp port 4500 -nn` while pinging `10.200.0.2`. You will observe ESP-encapsulated payloads hiding the underlying Geneve headers and inner IP data[cite: 1].

---

## 🛠️ Automated Provisioning Features

* **Netplan Persistence**: Internal IP addresses (`10.100.0.0/24` and `10.200.0.0/24`) are persisted via `/etc/netplan/60-ovs-internals.yaml`[cite: 1].
* **Flow Persistence**: Custom systemd unit `/etc/systemd/system/ovs-restore-flows.service` reloads saved flows from `/etc/openvswitch/br0-flows.txt` upon reboot[cite: 1].
* **Automated IPsec Service**: Systemd enables `openvswitch-ipsec` to manage StrongSwan tunnel key negotiations automatically using standard OVS options (`options:psk=...`)[cite: 1].

## **Watch a quick demo at**

https://www.youtube.com/watch?v=_12egWUnd8I 
