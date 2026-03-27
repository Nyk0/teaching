# Lab: Automated Virtual Machine Provisioning with Cloud-Init

## Objective

In this lab, students will learn how to automatically provision virtual machines using **Cloud-Init**. Two different initialization methods will be explored:

- **Method 1:** Cloud-Init data served over **HTTP**
- **Method 2:** Cloud-Init data provided through a **seed ISO**

By the end of the lab, students should be able to:

- understand the role of Cloud-Init in VM automation,
- prepare valid `user-data` and `meta-data` files,
- boot a VM with QEMU/KVM using Cloud-Init,
- compare the advantages and limitations of the two provisioning methods.

---

## Prerequisites

Recommended tools:

```bash
sudo apt update
sudo apt install -y qemu-system-x86 qemu-utils mkisofs python3
```

---

## Background

**Cloud-Init** is a widely used initialization framework for cloud instances and virtual machines. It allows the automatic configuration of a system at first boot, including:

- hostname configuration,
- user creation,
- SSH key injection,
- package installation,
- command execution.

In this lab, the VM will use the **NoCloud** datasource, which is convenient for local experiments and teaching.

The NoCloud datasource can obtain its configuration in several ways. In this lab, we focus on:

- **HTTP retrieval**, where the VM downloads its initialization files from a web server,
- **local ISO attachment**, where the initialization files are packaged into a virtual CD-ROM.

---

# Part 1 — Preparing the Environment

## 1. Download a cloud image

Download a Linux cloud image, for example an Ubuntu Server Cloud Image:

```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

You may rename it for convenience:

```bash
mv noble-server-cloudimg-amd64.img vm-base.qcow2
```

---

## 2. Create a Cloud-Init configuration directory

```bash
mkdir -p cloudinit-lab
cd cloudinit-lab
```

Create the following files.

### `user-data.yaml`

```yaml
#cloud-config
password: azerty
chpasswd:
  expire: False
ssh_pwauth: True
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABA...
```

### `meta-data.yaml`

```yaml
instance-id: vm1
local-hostname: cloudinit-vm
```

Then, you can create the seed.img:

```bash
cloud-localds my-seed.img user-data.yaml meta-data.yaml
```

Launch the virtual machine:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -smp 2 \
  -cpu host \
  -drive file=vm-base.qcow2,format=qcow2,if=virtio \
  -drive format=raw,file=seed.img,if=virtio \
  -nic user,model=virtio-net-pci,hostfwd=tcp:127.0.0.1:2222-:22 \
  -display none \
  -vnc 127.0.0.1:1 \
  -audiodev none,id=noaudio
```

Wait for installation and test with your ssh key:

```bash
ssh -p 2222 -i key_vm ubuntu@127.0.0.1
```

Logout from your virtual machine ans expand the qcow2 disk:

```bash
qemu-img resize vm-base.qcow2 +10G
sudo fdisk /dev/vda
sudo growpart /dev/vda
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
df -h
```
---

# Part 2 — Method 1: Cloud-Init via HTTP

## Principle

In this method, the VM is instructed to retrieve its Cloud-Init configuration files from an HTTP server. The NoCloud datasource can be pointed to a URL at boot time.

This is useful when:

- you want to centralize configuration files,
- you want to modify VM initialization without rebuilding an ISO,
- you want to provision multiple VMs from one server.

---

## 1. Start a simple HTTP server

From the directory containing `user-data` and `meta-data`:

```bash
python3 -m http.server 8000
```

Keep this terminal open.

---


## 2. Create configuration files

Create the following files under http location.

### `user-data.yaml`

```yaml
#cloud-config
autoinstall:
  version: 1

  locale: fr_FR.UTF-8
  keyboard:
    layout: fr

  identity:
    hostname: ubuntu-cloud
    username: devops
    password: "XXX"

  shutdown: poweroff

  ssh:
    install-server: true
    allow-pw: true
    authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAA...

  storage:
    layout:
      name: direct

  packages:
    - vim
```

### `meta-data.yaml`

```yaml
instance-id: ubuntu-cloud-001
local-hostname: ubuntu-cloud
```

---

## 3. Boot the VM with QEMU

In another terminal, run:

```bash
qemu-system-x86_64 \
  -enable-kvm
  -m 4096
  -smp 2
  -cpu host
  -drive file=vm1.qcow2,format=qcow2,if=virtio
  -kernel ./vmlinuz
  -initrd ./initrd
  -append "autoinstall ds=nocloud-net;s=http://10.0.2.2:8000/"
  -nic user,model=virtio-net-pci
  -display none
  -vnc 127.0.0.1:1
  -audiodev none,id=noaudio
```

Wait a minute ... Where are vmlinuz and initrd ? Dig in original ISO image, for instance ubuntu-24.04.4-live-server-amd64.iso.

### Important note

This approach requires booting with a compatible **kernel** and **initrd**, and passing the Cloud-Init datasource parameters through the kernel command line.

In a local QEMU environment:

- `10.0.2.2` is typically the host address as seen from the guest when using QEMU user-mode networking,
- the HTTP directory must expose:
  - `http://10.0.2.2:8000/user-data`
  - `http://10.0.2.2:8000/meta-data`

### Alternative note

If students use an installation workflow or a custom image-building process, the HTTP datasource can also be embedded in the boot parameters of that workflow.

---

## 3. Verify the provisioning

Log into the VM and check:

```bash
cloud-init status --long
```

Also inspect logs:

```bash
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log
```

---

## Questions

1. What is the purpose of the `user-data` file?
2. What is the purpose of the `meta-data` file?
3. Why must the VM be able to reach the HTTP server during first boot?
4. What is the meaning of `ds=nocloud-net`?
5. What happens if the HTTP server is unavailable at boot time?
