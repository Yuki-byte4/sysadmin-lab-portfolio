# Virtualization and Docker Lab

## Risks of Running Multiple Virtual Machines

Running multiple virtual machines (VMs) introduces several security and performance risks.

- **VM Escape:** A compromised VM may exploit a vulnerability in the hypervisor to access the host system or neighbouring VMs.
- **Side-Channel Attacks:** VMs sharing physical hardware may be exposed to attacks that use CPU cache or resource usage patterns to infer sensitive information.
- **Host Compromise:** If the host system is compromised, all VMs running on it may also be affected.
- **Lateral Movement:** Poorly monitored virtual network traffic can allow attackers to move between VMs and spread across the internal network.

### 1. Overcommitment

A hypervisor can allocate more virtual CPUs (vCPU) and virtual memory (vRAM) than the physical hardware actually provides. This works because VMs usually do not use all their allocated resources simultaneously.

However, when several VMs require high resources at the same time, CPU contention and increased latency can occur. Excessive memory overcommitment may force the host to use swap space, significantly reducing performance or causing VM processes to be terminated.

### 2. Virtual Switch

A virtual switch is a software-based networking device inside a hypervisor. It forwards Ethernet traffic between virtual network adapters without requiring physical cables.

It can also connect virtual machines to external networks through the host's physical network adapters. Virtual switches support network isolation through virtual port groups and VLAN tagging.

### 3. Nested Virtualization

Nested virtualization means running a hypervisor or virtualization platform inside an already virtualized machine.

It is useful for:
- Building virtual testing environments.
- Running nested hypervisors and virtualization platforms.
- Creating isolated development and testing labs.
- Running container environments that require hardware virtualization support.

In VMware Workstation, nested virtualization can be enabled by powering off the VM, opening **Settings → Processors**, and selecting **Virtualize Intel VT-x/EPT or AMD-V/RVI**.

---

# Virtualization

## (A) Proxmox VE Installation

I downloaded the Proxmox Virtual Environment ISO and created a new virtual machine.

Before starting the installation, I configured the virtual hardware with two network adapters:

- **NAT:** Provides internet connectivity.
- **Host-only:** Allows internal communication between the virtual machines and the host.

The installation was completed using the Proxmox installer.

**Screenshots:**
- [Proxmox installer network configuration](screenshots/proxmox-installer.png)
- [Proxmox web interface](screenshots/proxmox-ui.png)

## (B) Network Configuration

After installing Proxmox, I checked the available network interfaces and configured the host-only adapter.

The host-only network address was identified from the host's network configuration. For documentation purposes, the following commands use example private IP addresses.

```bash
ip link set nic1 up
ip addr add 192.168.10.10/24 dev nic1
```

Initially, ping requests from Proxmox to the Windows host failed because the Windows firewall was blocking incoming ICMP traffic.

To investigate the issue, I checked the Windows firewall rules using PowerShell and enabled the required connectivity rule.

```powershell
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing"
```

**Screenshots:**
- [Proxmox IP address configuration](screenshots/proxmox-ip-addr.png)
- [Network interfaces configuration](screenshots/proxmox-interfaces.png)
- [Ping failing before firewall configuration](screenshots/ping-fail.png)
- [Ping working after firewall configuration](screenshots/ping-ok.png)

## (C) Debian VM Inside Proxmox

After creating the Debian virtual machine inside Proxmox, I configured its virtualization and networking settings.

In my environment, I had to disable KVM hardware virtualization in the VM options to resolve a compatibility issue.

**Screenshot:**
- [KVM hardware virtualization settings](screenshots/kvm-disabled.png)

Next, I configured a second network adapter for host-only communication through the Proxmox web interface.

The configuration steps were:

1. Navigate to **pve → System → Network**.
2. Create a new Linux bridge named `vmbr1`.
3. Set `nic1` as the bridge port.
4. Click **Apply Configuration**.
5. Verify the bridge using `bridge link` in the Proxmox shell.
6. Select the Debian VM and navigate to **Hardware → Add → Network Device**.
7. Assign the new network device to `vmbr1`.

**Screenshots:**
- [vmbr1 bridge configuration](screenshots/vmbr1.png)
- [Second network adapter on Debian VM](screenshots/vm-network-device.png)
- [Bridge link command output](screenshots/bridge-link.png)

The second Debian network interface was initially down and had no IP address. I brought it up and assigned an example address from the host-only network.

```bash
sudo ip link set ens34 up
sudo ip addr add 192.168.10.20/24 dev ens34
```

After configuring both network adapters, I tested connectivity between the Debian VM, the Proxmox server, the host system, and the internet.

**Screenshot:**
- [Debian network interface configuration](screenshots/debian-ip.png)

## MariaDB Installation and Configuration

I installed MariaDB on the Debian virtual machine:

```bash
sudo apt install mariadb-server
```

After installation, I configured MariaDB to accept connections only through the designated host-only network interface.

This ensured that database access remained restricted to the intended internal network rather than being exposed to the external network.

**Screenshot:**
- [MariaDB installation and service status](screenshots/mariadb-install.png)

---

# Docker

## (A) Windows Virtual Machine

I created an additional Windows virtual machine using a Windows ISO.

The VM was configured with a second host-only network adapter to allow communication with the Debian database server.

I also enabled nested virtualization through:

**Settings → Processors → Virtualize Intel VT-x/EPT or AMD-V/RVI**

**Screenshot:**
- [Windows VM settings](screenshots/windows-vm-settings.png)

## (B) Docker Desktop Installation

I installed Docker Desktop inside the Windows virtual machine and configured Linux container support.

To enable the required Windows Subsystem for Linux (WSL) functionality, I executed the following command in an administrator PowerShell terminal:

```powershell
wsl --install
```

After completing the installation, I verified that Docker Desktop was running correctly.

**Screenshot:**
- [Docker Desktop installation](screenshots/docker-desktop.png)

## (C) phpMyAdmin Container

The next step was to deploy phpMyAdmin using Docker.

The goal was to establish communication between the phpMyAdmin container running inside the Windows VM and the MariaDB server installed on the Debian VM.

This allowed me to access and manage the database through a web-based interface while keeping database communication within the host-only network.

