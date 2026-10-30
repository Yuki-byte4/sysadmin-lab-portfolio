# Lab 0: VM Setup, Virtual Network and Snapshots


## Contents
1. [Preparation](#1-preparation)
2. [Virtual network topology](#2-virtual-network-topology)
3. [Snapshots](#3-snapshots)
4. [Failure scenario](#4-failure-scenario)
5. [Problems and fixes](#5-problems-and-fixes)

---

## 1. Preparation

1. Install VMware Workstation.
2. Install Debian (the version matching your PC) and create a new Workstation
   VM, **VM1.1**.
3. Make a **linked clone** of VM1 (**VM2**). A linked clone shares the parent's
   disk and only stores the differences, so it is fast to create and small.
4. Add a Windows 11 Workstation VM (**VM3.1**) from a Windows ISO.

---

## 2. Virtual network topology

Several changes were needed on the VMs:

1. Create a **host-only VMnet** in VMware's **Virtual Network Editor**
   (top menu), add a new network and configure it.
2. Give each VM an adapter on that network and assign an IP address and subnet
   mask.

**Debian:**

    sudo ip addr add <IP>/24 dev <IFACE>

**Windows:** Settings → Network & Internet → Advanced network settings →
VMnet2 → Edit → double-click IPv4 → enter the IP address and subnet mask.

All VMs are on the same internal network, so VM1.1 reaches VM2 and VM3.1, VM2
reaches VM1 and VM3, and VM3.1 reaches VM2 and VM1.1. The connection is stable
and each host answers `ping`:

    Debian:   ping -c 4 <IP>
    Windows:  ping <IP>

![VM1.1 pinging VM2](screenshots/ping-vm1-to-vm2.png)
![VM2 pinging VM1.1](screenshots/ping-vm2-to-vm1.png)
![Debian pinging Windows](screenshots/ping-debian-to-windows.png)
![Windows pinging Debian](screenshots/ping-windows-to-debian.png)

### Windows firewall and ping
For ping to work towards Windows, I had to deactivate the firewall, because it
was dropping ICMP packets. After that everything worked.

(corrected) Turning the firewall off is fine in an isolated lab, but a safer
fix is to allow only ICMP, in an administrator PowerShell:

    Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"

---

## 3. Snapshots

A snapshot saves the state of a VM at a given moment, so you can return to it
later. It is manual, so you have to take it yourself (**VM → Snapshot → Take
Snapshot**).

(corrected) A snapshot is not a real backup. It is stored next to the VM's own
disk, so if the disk is lost the snapshots are lost too. It is a quick
restore point, not a replacement for backups.

![Snapshot manager](screenshots/snapshot-manager.png)

---

## 4. Failure scenario

Snapshots matter in real setups because you never know when something will go
wrong, so it is always best to have a recovery plan ready. To test this I set up
a failure scenario:

1. I turned the Windows firewall **on** again.
2. I removed the Windows VM (VM3.1).
3. As the first screenshot shows, the host became completely unreachable.
4. I then restored the system quickly from my saved snapshot (second
   screenshot).

![Host unreachable after the failure](screenshots/failure-unreachable.png)
![Recovered from the snapshot](screenshots/failure-recovered.png)

---

## 5. Problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| Ping to Windows fails | Windows firewall drops ICMP | Allow the ICMP rule (or disable the firewall in a lab) |
| Second adapter has no address | Interface down, no DHCP on host-only network | `ip link set <IFACE> up`, then `ip addr add` |
| Host unreachable after the failure test | Firewall on and the VM removed | Revert to the saved snapshot |

## What I learned
- A host-only network connects VMs to each other and to the host without
  exposing them to the outside network.
- A firewall can look like a network problem, so check it when ping fails.
- Snapshots give a fast way back after a mistake, but they are not backups.
