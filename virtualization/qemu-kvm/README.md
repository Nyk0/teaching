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

Create a regular user with sudo privileges :

```bash
useradd -m -s /bin/bash user
passwd user
apt-get install sudo
usermod -a -G sudo user
id user
```

Install the required packages:

```bash
sudo apt install -y qemu-system-x86 qemu-utils
```

Add regular user to kvm group :

```bash
ls -al /dev/kvm
sudo usermod -a -G kvm user
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

Create a virtual disk image in `qcow2` format with a size of **10 GiB**:

```bash
qemu-img create -f qcow2 vm1.qcow2 10G
```

Display information about the disk image:

```bash
qemu-img info vm1.qcow2
```

### Questions

1. What is the size of the virtual disk?
2. What is the disk format?
3. Is the full 10 GiB immediately allocated on the host?

---

## Part 2 — Start the Virtual Machine

You will now boot a VM from the Debian ISO image.

Download ISO installation media :

```bash
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.3.0-amd64-netinst.iso
```

Use the following command:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -drive file=vm1.qcow2,format=qcow2,if=virtio \
  -cdrom debian-13.3.0-amd64-netinst.iso \
  -boot d \
  -nic user,model=virtio-net-pci \
  -display none \
  -vnc 127.0.0.1:1 \
  -audiodev none,id=noaudio
```

### Questions

1. Why do we use `-enable-kvm`?
2. Why is the disk attached with `if=virtio`?
3. What is the purpose of `-boot d`?
4. Why is VNC useful in this context?

---

## Part 3 — Connect to the Installer with VNC

The VM exports its display through VNC on display `127.0.0.1:1`.

Install a vnc client:

```bash
sudo apt install xtightvncviewer
```

Connect to it with a VNC client using:

```bash
xtightvncviewer localhost:5901
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

Restart the guest:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -drive file=vm1.qcow2,format=qcow2,if=virtio \
  -nic user,model=virtio-net-pci \
  -display none \
  -vnc 127.0.0.1:1 \
  -audiodev none,id=noaudio
```

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
sudo shutown -h now
```

Wait until the QEMU process exits.

### Question

Why is it important to shut down the VM cleanly before inspecting the disk image offline?

---

## Part 6 — Explore the Disk Image with NBD

Now that the VM is stopped, you will inspect the disk image manually from the host.

### Requirements

Install nbd tools:

```bash
sudo apt install nbd-client fdisk parted kpartx
```

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
2. Do you see a Linux root partition?
3. Is the partition layout the same as what you selected during installation?

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

## Part 8 — Inject SSH key for user

Tips:

Generate SSH keys (on host):

```bash
ssh-keygen -t rsa -b 4096 -f vm_key
```

Public part must reside in **~/.ssh/authorized_keys**.

Adjust **/etc/ssh/sshd_config** to allow only public key authentication.

## Part 9 — Clean Up

Unmount the mounted partitions:

```bash
sudo umount /mnt/vm1
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
11. What is the difference between the bootloader files and the Linux kernel files?

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
  -display none \
  -vnc 127.0.0.1:1 \
  -audiodev none,id=noaudio
```

Then try to connect from the host with:

```bash
ssh -i vm_key -p 2222 user@127.0.0.1
```

### Extension Questions

1. What does `hostfwd=` do?
2. Why is this useful in a NAT configuration?
3. What changes from the guest point of view?
