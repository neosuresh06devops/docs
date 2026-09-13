# Troubleshooting Guide: Kernel Panic - VFS: Unable to mount root fs

## Metadata
* **Author:** neosuresh06devops
* **Component:** Linux Kernel / Bootloader
* **Severity:** High (System Unbootable)
* **Affected OS:** Ubuntu 22.04 LTS (Kernel `5.15.0-191-generic`)

---

## 1. Issue Detected

### Symptoms
* The virtual machine fails during early boot stages.
* Boot console halts with the following trace:
  ```text
  [    1.298539] panic+0x15c/0x33b
  [    1.298587] mount_block_root+0x144/0x1dd
  [    1.298648] mount_root+0x10c/0x11c
  [    1.299274] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
Impact:
The operating system cannot locate or load the root filesystem (/).
Standard userspace init processes (systemd) fail to initialize.

2. Root Cause Analysis (RCA)
Primary Cause: The initial RAM filesystem (initramfs) image corresponding to the target kernel version (initrd.img-5.15.0-191-generic) is either missing, truncated, or corrupted.

Contributing Factors:

Insufficient Disk Space in /boot: Unattended kernel upgrades triggered while /boot or / had 100% inode or disk space utilization, resulting in an incomplete update-initramfs run.

Interrupted Package Upgrades: The VM was abruptly rebooted or powered off during an active apt upgrade or dpkg transaction.

Missing Storage Drivers: Storage controller modules (such as VMware mptspi or vmw_pvscsi) were omitted during image generation.


3. Recovery & Resolution
Phase 1: Temporary Bypass (GRUB Fallback)
Reset the VM via the VMware console (CTRL+ALT+DEL or Power Reset).

Tap Esc (or hold Shift) immediately to access the GRUB Boot Menu.

Select Advanced options for Ubuntu.

Choose the previous working kernel version (e.g., an earlier 5.15.0-xx-generic entry).

Boot into the OS and log in.

Phase 2: Disk Remediation
Verify available storage on /boot and /:
df -h /boot /

If disk usage is at or near 90%, clean the package cache and remove old kernels:

# Clean APT cache archives
sudo apt-get clean

# Purge unused kernels and dependencies
sudo apt-get autoremove --purge -y

Phase 3: Initramfs Rebuild & GRUB Update
Regenerate the RAM disk image for all installed kernels and update the bootloader:
# Rebuild initramfs across all detected kernel versions
sudo update-initramfs -u -k all

# Refresh GRUB configuration
sudo update-grub


4. Verification
a. Check that the new initramfs file size is valid and non-zero:
ls -lh /boot/initrd.img-*

b. Reboot the system normally:
   sudo reboot
c. Confirm that the VM boots directly into the updated kernel:
uname -r


5. Preventative Measures
Automate Kernel Cleanup: Keep /etc/apt/apt.conf.d/01autoremove enabled to retain only the running kernel and one previous fallback version.

Storage Monitoring: Set up disk alerts for /boot usage exceeding 85%.

Pin Critical Kernels: In production environments, hold kernel packages until scheduled maintenance:

Bash
sudo apt-mark hold linux-image-generic linux-headers-generic
