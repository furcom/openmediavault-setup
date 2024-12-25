## Info:
This guide explains how to setup OMV in UEFI mode in Proxmox with PCIe passthrough of a SATA controller.
##### Be sure IOMMU is supported and enabled on your Proxmox host: [https://pve.proxmox.com/wiki/PCI_Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
---
### On Proxmox-Host:
1. Edit Grub:
```
vim /etc/default/grub
```
2. After `quiet` in line `GRUB_CMDLINE_LINUX_DEFAULT=` add this:
- On AMD:
```
amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction
```
- On INTEL:
```
intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction
```
3. Save changes:
```
update-grub && reboot
```
---
### VM hardware (CPU and RAM is up to you):
1. BIOS: OVMF (UEFI)
2. Machine: q35
3. Disk: VirtIO SCSI Block
4. RAM: Choose your RAM but `disable ballooning` !!!
5. Add `PCI Device` with following options:
- RAW Device: ✅ <choose_your_device> (I'm adding my SATA controller here)
- All functions: ✅
- ROM-Bar: ✅
- PCI-Express: ✅
### VM options:
- Hotplus: Uncheck `Disk`
---
### OMV setup:
1. Install minimal Debian without GUI, otherwise OMV won't work !!!
2. Install following (as root):
```
apt-get install --yes gnupg
```
```
wget --quiet --output-document=- https://packages.openmediavault.org/public/archive.key | gpg --dearmor --yes --output "/usr/share/keyrings/openmediavault-archive-keyring.gpg"
```
3. Add repo:
```
vim /etc/apt/sources.list.d/openmediavault.list
```
```
deb [signed-by=/usr/share/keyrings/openmediavault-archive-keyring.gpg] https://packages.openmediavault.org/public sandworm main
```
4. Install OMV (as root):
```
export LANG=C.UTF-8
export DEBIAN_FRONTEND=noninteractive
export APT_LISTCHANGES_FRONTEND=none
apt-get update
apt-get --yes --auto-remove --show-upgraded --allow-downgrades --allow-change-held-packages --no-install-recommends --option DPkg::Options::="--force-confdef" --option DPkg::Options::="--force-confold" install openmediavault
```
5. Execute following commands:
```
omv-confdbadm populate
```
```
omv-salt deploy run systemd-networkd
```
---
### If system/package updates via WebGUI gives you “Error 500 …”
1. Execute this and go through the options:
```
omv-firstaid
```
2. Check for updates and install if available:
```
apt update && apt upgrade -y
```
3. Reboot OMV
