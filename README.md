# Info:
This guide explains how I setup OMV in UEFI mode in Proxmox with a PCIe SATA controller via Proxmox PCI passthrough.
##### Be sure IOMMU is supported by your hardware and enabled on your Proxmox host: [https://pve.proxmox.com/wiki/PCI_Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
  
---
  
## 1. - ToDo on Proxmox-Host:
### 1.1 - Edit Grub:
```
vim /etc/default/grub
```
After `quiet` in line `GRUB_CMDLINE_LINUX_DEFAULT=` add this:  
- AMD:
```
amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction
```
- INTEL:
```
intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction
```
Save Grub changes and reboot:
```
update-grub && reboot
```
  
### 1.2 - Prevent Proxmox from renaming network interface:
Create systemd-link:
```
vim /etc/systemd/network/10-eth0.link
```
Add this into link:
```
[Match]
MACAddress=<mac_address_proxmox_network_interface>
Type=ether

[Link]
Name=eth0
```
Reload systemd
```
systemctl daemon-reload
```
Dont forget to rename interface in network config, too:
```
vim /etc/network/interfaces
```
Network config example:
```
auto lo
iface lo inet loopback
iface eth0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 1.2.3.4/24
        gateway 1.2.3.1
        bridge-ports eth0
        bridge-stp off
        bridge-fd 0
source /etc/network/interfaces.d/*
```
  
### 1.3 - Prevent Proxmox from loading SATA controller's drivers:
Add modprobe config:
```
vim /etc/modprobe.d/vfio.conf
```
Add this into the config (`ids=03:00.0,0000:03:00.0` is my SATA controller's ID. Change this to yours)
```
# VFIO for PCI-devices
options vfio_iommu_type1 allow_unsafe_interrupts=1
options vfio-pci ids=03:00.0,0000:03:00.0 

# Deactivate/blacklist AHCI, if it gets loaded automatically
blacklist ahci
blacklist ata_piix
blacklist libata
```
Apply changes and reboot:
```
update-initramfs -u -k all && reboot
```
  
### 1.4 - VM hardware (CPU and RAM is up to you):
1. BIOS: `OVMF (UEFI)`
2. Machine: `q35`
3. Disk: `VirtIO SCSI Block`
4. RAM: `Choose your RAM but disable ballooning !!!`
5. Add `PCI Device` with following options:
- RAW Device: `<choose_your_device> (I'm adding my SATA controller here)`
- All functions: ✅
- ROM-Bar: ✅
- PCI-Express: ✅
  
### 1.5 - VM options:
- Hotplug: Uncheck `Disk`
  
---
  
## 2. - OMV Setup (Every step is done as root user):
  
### 2.1 - Install minimal Debian without GUI, otherwise OMV won't work !!!
  
### 2.2 - Install gnupg and keyring:
```
apt-get install --yes gnupg
```
```
wget --quiet --output-document=- https://packages.openmediavault.org/public/archive.key | gpg --dearmor --yes --output "/usr/share/keyrings/openmediavault-archive-keyring.gpg"
```
### 2.3 - Add repo:
```
vim /etc/apt/sources.list.d/openmediavault.list
```
Add this into repo file:
```
deb [signed-by=/usr/share/keyrings/openmediavault-archive-keyring.gpg] https://packages.openmediavault.org/public sandworm main
```
### 2.4 - Install OMV:
```
export LANG=C.UTF-8
export DEBIAN_FRONTEND=noninteractive
export APT_LISTCHANGES_FRONTEND=none
apt-get update
apt-get --yes --auto-remove --show-upgraded --allow-downgrades --allow-change-held-packages --no-install-recommends --option DPkg::Options::="--force-confdef" --option DPkg::Options::="--force-confold" install openmediavault
```
### 2.5 - Execute following commands:
```
omv-confdbadm populate
```
```
omv-salt deploy run systemd-networkd
```
  
### 2.6 - If system/package updates via WebGUI gives you “Error 500 …”
1. Execute this and go through the options:
```
omv-firstaid
```
2. Check for updates and install if available:
```
apt update && apt upgrade -y
```
3. Reboot OMV
  
---
  
### 3. - Enabling ZFS filesystem in OMV
3.1 Install `omv-extras`:
```
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```
3.2 Install `openmediavault-zfs`:
- Go to `System` > `Plugins` > Search for `zfs` > Install `openmediavault-zfs` plugin
- Reboot system
