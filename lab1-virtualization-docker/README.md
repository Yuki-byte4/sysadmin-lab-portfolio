# Lab 1: Virtualization and Docker


## Concepts

1. **Risks of running multiple VMs.** A breached VM can bypass the hypervisor
   (VM escape) to reach the host or neighbouring machines. Shared host hardware
   opens up side-channel vulnerabilities, where a malicious VM reads memory or
   cache usage patterns. Compromising the host compromises every hosted VM, and
   unmonitored internal virtual network traffic lets attacks spread laterally.

2. **Overcommitment.** A hypervisor can allocate more vCPU and vRAM than the
   physical hardware has, assuming VMs will not hit peak usage at the same time.
   When several VMs demand high resources at once, CPU contention and high
   latency occur. Extreme memory overcommitment forces the host to swap, which
   slows everything down or makes the host terminate VM processes.

3. **Virtual switch.** A software device inside the hypervisor that forwards
   ethernet traffic between virtual network adapters of local VMs without
   physical cables. It connects to the host's physical NICs through uplink ports
   to bridge VM traffic with external networks, and it enforces isolation with
   virtual port groups and VLAN tagging.

4. **Nested virtualization.** Running a hypervisor or container platform inside
   an already virtualized guest. It is used for virtual test labs, for Docker
   environments that need low-level hardware access, and for sandboxing. In
   VMware Workstation: power off the VM, open Settings, select Processors and
   tick "Virtualize Intel VT-x/EPT or AMD-V/RVI".

## Virtualization

### (A) Proxmox VE installation
I downloaded the Proxmox Virtual Environment ISO and created a new virtual
machine. Before running Proxmox (which is based on Debian), I configured the
virtual hardware with two network adapters: one **NAT** and one **Host-only**
(for internal communication with the host).

![Proxmox installer network configuration](screenshots/proxmox-installer.png)
![Proxmox web interface](screenshots/proxmox-ui.png)

### (B) Network configuration
I checked the network connections on the server. `nic1` had to be brought up and
given an address in the host-only range, which I found by running `ipconfig` in
cmd on the host.

    ip link set nic1 up
    ip addr add 192.168.229.10/24 dev nic1

![ip addr output on Proxmox](screenshots/proxmox-ip-addr.png)
![/etc/network/interfaces](screenshots/proxmox-interfaces.png)

Everything looked fine, but the ping from the VM to the host failed. Windows has
a strict firewall, so ICMP requests do not go through. I had to enable a
firewall rule. In an admin PowerShell, the first command shows the rules and the
second enables them:

    Get-NetFirewallRule -DisplayGroup "File and Printer Sharing"
    Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"

![Ping failing before the rule](screenshots/ping-fail.png)
![Ping working after the rule](screenshots/ping-ok.png)

### (C) Debian VM inside Proxmox
After creating the Debian VM inside Proxmox it has to be configured. In some
cases VMs do not run because KVM is enabled, so I disabled it in the VM options.

![KVM hardware virtualization disabled](screenshots/kvm-disabled.png)

I then added a second network adapter (host-only) besides the NAT one.
Everything was done from the Proxmox GUI:

1. Left menu: **pve → System → Network**.
2. Create a new bridge `vmbr1` with bridge port `nic1` (host-only on Proxmox).
3. Click **Apply Configuration** (new interfaces must be applied manually).
4. Check from the pve shell with `bridge link`.
5. Select the Debian VM → **Hardware → Add → Network Device**, bridge `vmbr1`.

![vmbr1 bridge](screenshots/vmbr1.png)
![Second network device on the VM](screenshots/vm-network-device.png)
![bridge link output](screenshots/bridge-link.png)

In my case the second interface in Debian was down and had no address, so I
brought it up and gave it an address:

    sudo ip link set ens34 up
    sudo ip addr add 192.168.229.20/24 dev ens34

Then I could check connectivity by pinging the internet, the host and the server.

![Debian interface configuration](screenshots/debian-ip.png)

### MariaDB
I installed the database with `sudo apt install mariadb-server` and made sure it
is only accessible from the second (host-only) network.

![MariaDB installation and status](screenshots/mariadb-install.png)

## Docker

### (A) Windows VM
I created another VM outside Proxmox with a Windows ISO, configured it like the
previous one (second host-only adapter) and enabled "Virtualize Intel VT-x/EPT
or AMD-V/RVI" in the Processors section.

![Windows VM settings](screenshots/windows-vm-settings.png)

### (B) Docker Desktop
I installed Docker Desktop in the Windows VM and configured Linux container
support (`wsl --install` in PowerShell).

![Docker Desktop installed](screenshots/docker-desktop.png)

### (C) phpMyAdmin container
I started a container from the official phpMyAdmin
