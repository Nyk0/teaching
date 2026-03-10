# Lab Practical — Introduction to KVM/QEMU on Debian Trixie

## Overview

In this lab, you will create and install a virtual machine manually with **QEMU/KVM** on a **Debian Trixie** host.

The goal is to understand the basic building blocks of virtualization without using higher-level tools such as `libvirt` or `virt-manager`.

You will:

- create a virtual disk in `qcow2` format,
- boot a VM from an ISO image,
- use a **NAT** network configuration,
- access the VM installer through **VNC**,
- complete an interactive installation,
- stop the VM,
- inspect the content of the virtual disk manually using **NBD**.

---

## Learning Objectives

At the end of this lab, you should be able to:

- explain the role of QEMU and KVM,
- create a virtual disk image with `qemu-img`,
- launch a VM from the command line,
- connect to a VM through VNC,
- identify the effect of QEMU user-mode networking,
- inspect a guest disk image offline,
- identify partitions and filesystems inside a VM image.

---

## Prerequisites

You need:

- a Debian Trixie host,
- access to a terminal on the host,
- hardware virtualization available on the machine,
- a Debian installation ISO image,
- the required packages installed.

Install the required packages:

```bash
sudo apt update
sudo apt install -y qemu-system-x86 qemu-utils qemu-system-gui \
                    ovmf nbd-client fdisk parted kpartx util-linux
```

---

## Working Directory

Create a dedicated working directory for this lab:

```bash
mkdir -p ~/lab-qemu
cd ~/lab-qemu
```

You should store your files there.

Example:

```text
~/lab-qemu/
├── debian-13.iso
└── vm1.qcow2
```

---

## Part 1 — Create a Virtual Disk

Create a virtual disk image in `qcow2` format with a size of **20 GiB**:

```bash
qemu-img create -f qcow2 vm1.qcow2 20G
```

Display information about the disk image:

```bash
qemu-img info vm1.qcow2
```

### Questions

1. What is the size of the virtual disk?
2. What is the disk format?
3. Is the full 20 GiB immediately allocated on the host?

---

## Part 2 — Start the Virtual Machine

You will now boot a VM from the Debian ISO image.

Use the following command:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -drive file=vm1.qcow2,format=qcow2,if=virtio \
  -cdrom debian-13.iso \
  -boot d \
  -nic user,model=virtio-net-pci \
  -vnc :1
```

### Understanding the Command

Make sure you understand the role of the following options:

- `-enable-kvm`
- `-m 2048`
- `-smp 2`
- `-cpu host`
- `-drive file=vm1.qcow2,format=qcow2,if=virtio`
- `-cdrom debian-13.iso`
- `-boot d`
- `-nic user,model=virtio-net-pci`
- `-vnc :1`

### Questions

1. Why do we use `-enable-kvm`?
2. Why is the disk attached with `if=virtio`?
3. What is the purpose of `-boot d`?
4. Why is VNC useful in this context?

---

## Part 3 — Connect to the Installer with VNC

The VM exports its display through VNC on display `:1`.

Connect to it with a VNC client using:

```text
localhost:5901
```

Once connected:

- start the interactive Debian installation,
- follow the normal installation process,
- create at least one user account,
- install the base system,
- reboot the VM at the end of the installation.

When the installation is complete, log in once to verify that the system boots correctly.

---

## Part 4 — Inspect the Guest Network

Once logged into the guest system, run the following commands:

```bash
ip a
ip route
cat /etc/resolv.conf
```

Observe the network configuration.

### Questions

1. What IP address does the guest receive?
2. Is it a private address?
3. Can the guest access the Internet?
4. Can the host directly reach the guest?
5. Why is this networking mode called NAT or user-mode networking?

---

## Part 5 — Shut Down the VM

When you have finished inspecting the guest, shut it down cleanly from inside the VM:

```bash
sudo poweroff
```

Wait until the QEMU process exits.

### Question

Why is it important to shut down the VM cleanly before inspecting the disk image offline?

---

## Part 6 — Explore the Disk Image with NBD

Now that the VM is stopped, you will inspect the disk image manually from the host.

### Step 1 — Load the NBD module

```bash
sudo modprobe nbd max_part=16
```

Check that NBD devices are present:

```bash
ls /dev/nbd*
```

### Step 2 — Connect the disk image to `/dev/nbd0`

```bash
sudo qemu-nbd --connect=/dev/nbd0 vm1.qcow2
```

### Step 3 — Re-read the partition table

```bash
sudo partprobe /dev/nbd0
```

Now inspect the exported disk:

```bash
sudo fdisk -l /dev/nbd0
lsblk /dev/nbd0
```

### Questions

1. How many partitions are present?
2. Do you see an EFI partition?
3. Do you see a Linux root partition?
4. Is the partition layout the same as what you selected during installation?

---

## Part 7 — Mount a Partition Manually

Create a mount point:

```bash
sudo mkdir -p /mnt/vm1
```

Mount the root partition.

For example, if the root filesystem is `/dev/nbd0p2`:

```bash
sudo mount /dev/nbd0p2 /mnt/vm1
```

Inspect the filesystem:

```bash
ls -lah /mnt/vm1
find /mnt/vm1 -maxdepth 2 | head -100
```

### Questions

1. Can you identify a standard Linux root filesystem?
2. Which directories are present?
3. Which files or directories prove that Debian has been installed?
4. Can you locate `/etc`, `/boot`, `/home`, and `/var`?

---

## Part 8 — Inspect the EFI Partition (if present)

If the VM was installed in UEFI mode, you may have an EFI system partition.

Create a mount point:

```bash
sudo mkdir -p /mnt/efi
```

Mount the EFI partition, for example:

```bash
sudo mount /dev/nbd0p1 /mnt/efi
```

Inspect its contents:

```bash
find /mnt/efi -maxdepth 3 -type f
```

### Questions

1. Do you find an `EFI` directory?
2. Do you see files related to Debian boot?
3. What is the difference between the EFI partition and the Linux root partition?

---

## Part 9 — Clean Up

Unmount the mounted partitions:

```bash
sudo umount /mnt/vm1
sudo umount /mnt/efi 2>/dev/null || true
```

Disconnect the NBD device:

```bash
sudo qemu-nbd --disconnect /dev/nbd0
```

---

## Analysis Questions

Answer the following questions in your report.

### Virtualization Basics

1. What is the role of QEMU?
2. What is the role of KVM?
3. Why can QEMU be used without libvirt in this lab?

### Storage

4. What is the advantage of `qcow2`?
5. What is the difference between a disk image file and a block device?
6. Why is NBD useful for image inspection?

### Networking

7. Why is QEMU user networking convenient for a first lab?
8. What are its limitations compared to bridged networking?
9. Why is this setup called NAT?

### Boot and Filesystems

10. Where are the boot files located?
11. What is the role of the EFI partition?
12. What is the difference between the bootloader files and the Linux kernel files?

---

## Deliverables

You must submit:

1. the exact command used to create the disk image,
2. the exact QEMU command line used to boot the VM,
3. a screenshot of the Debian installation through VNC,
4. the output of:

```bash
qemu-img info vm1.qcow2
```

5. the output of:

```bash
sudo fdisk -l /dev/nbd0
lsblk /dev/nbd0
```

6. a short description of the partition layout,
7. a short explanation of the guest network configuration,
8. a short explanation of what you observed after mounting the guest filesystem.

---

## Optional Extension

If you finish early, modify the VM command line to add SSH port forwarding from the host to the guest.

Example:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -drive file=vm1.qcow2,format=qcow2,if=virtio \
  -nic user,model=virtio-net-pci,hostfwd=tcp:127.0.0.1:2222-:22 \
  -vnc :1
```

Then try to connect from the host with:

```bash
ssh -p 2222 <guest-user>@127.0.0.1
```

### Extension Questions

1. What does `hostfwd=` do?
2. Why is this useful in a NAT configuration?
3. What changes from the guest point of view?

---

## Conclusion

This lab introduced the most direct way to create and use a virtual machine with QEMU/KVM.

You have worked with:

- a manually created `qcow2` disk image,
- a VM booted from an ISO,
- a NAT-based virtual network interface,
- a VNC-exported graphical installation,
- and offline disk analysis through NBD.

This approach gives a concrete understanding of what a virtual machine really is: a process, a set of virtual devices, and a disk image file.
