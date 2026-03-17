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
cloud-localds my-seed.img user-data.yaml my-meta-data.yaml
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

## 2. Boot the VM with QEMU

In another terminal, run:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -drive file=vm1.qcow2,format=qcow2,if=virtio \
  -nic user,model=virtio-net-pci \
  -serial mon:stdio \
  -append "ds=nocloud-net;s=http://10.0.2.2:8000/" \
  -kernel /path/to/vmlinuz \
  -initrd /path/to/initrd
```

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

---

# Part 3 — Method 2: Cloud-Init via seed ISO

## Principle

In this method, the Cloud-Init files are packaged into a small ISO image and attached to the VM as a virtual CD-ROM.

This is useful when:

- no network access is available at boot time,
- you want a fully local and self-contained provisioning process,
- you want a simple and reproducible lab setup.

---

## 1. Create the seed ISO

Using `cloud-localds`:

```bash
cloud-localds seed.iso user-data meta-data
```

If `cloud-localds` is not available, use `genisoimage`:

```bash
genisoimage -output seed.iso -volid cidata -joliet -rock user-data meta-data
```

The volume label **must** be `cidata` for NoCloud detection.

---

## 2. Boot the VM with QEMU

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -smp 2 \
  -cpu host \
  -drive file=vm1.qcow2,format=qcow2,if=virtio \
  -drive file=seed.iso,format=raw,media=cdrom \
  -nic user,model=virtio-net-pci \
  -serial mon:stdio
```

In this case, no HTTP server is required.

---

## 3. Verify the provisioning

Inside the VM, run:

```bash
hostname
id student
ls -l /tmp/cloudinit-success
cat /etc/motd
cloud-init status --long
```

You can also inspect the mounted datasource and logs:

```bash
mount | grep -i cdrom
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log
```

---

## Questions

1. Why must the ISO volume label be `cidata`?
2. What is the main advantage of the seed ISO method?
3. In which situations is the seed ISO method more practical than HTTP?
4. Which method is easier to scale for many different VMs?
5. Which method is more robust in an isolated lab environment?

---

# Part 4 — Comparison of the Two Methods

Complete the following comparison table:

| Criterion | HTTP NoCloud | Seed ISO NoCloud |
|---|---|---|
| Requires network access at boot |  |  |
| Easy to update configuration |  |  |
| Fully self-contained |  |  |
| Convenient for many VMs |  |  |
| Simple offline setup |  |  |

Then answer:

1. Which method would you choose for a teaching lab with no Internet access?
2. Which method would you choose for dynamic provisioning of many VMs?
3. Which method is easier to debug?
4. Which method better reflects cloud-style provisioning?

---

# Part 5 — Additional Exercises

## Exercise A — Add a second user

Modify `user-data` to create another user named `adminlab` with sudo access.

---

## Exercise B — Install additional software

Add the following packages through Cloud-Init:

- `git`
- `tmux`
- `tree`

Verify that they are installed after boot.

---

## Exercise C — Run a custom startup command

Modify `runcmd` so that the VM writes the current date into `/root/provisioning-date.txt`.

---

## Exercise D — Change the hostname

Change the hostname to `cloud-lab-node1` and verify that the change is applied correctly.

---

# Part 6 — Troubleshooting Guide

If Cloud-Init does not work as expected, check the following:

## 1. Validate the YAML syntax

Incorrect indentation is a common source of errors.

## 2. Check Cloud-Init logs

```bash
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log
```

## 3. Check datasource detection

```bash
cloud-init query ds
```

## 4. Ensure first-boot behavior

Cloud-Init usually runs on first boot. If you reuse the same image, previous state may prevent rerunning.

To clean the state inside the VM:

```bash
sudo cloud-init clean
```

Or create a fresh QCOW2 overlay before each test.

## 5. Verify connectivity for HTTP mode

Inside the guest, confirm that the host HTTP server is reachable if the network is already up.

---

# Expected Deliverables

Students must submit:

1. their final `user-data` file,
2. their final `meta-data` file,
3. the command line used for the **HTTP** method,
4. the command line used for the **seed ISO** method,
5. a short comparison paragraph between the two methods,
6. screenshots or terminal outputs proving that Cloud-Init executed successfully.

---

# Assessment Criteria

Students will be evaluated on:

- correctness of the Cloud-Init files,
- successful boot and provisioning of the VM,
- understanding of the two NoCloud delivery methods,
- ability to verify and troubleshoot the initialization,
- clarity of the comparison and conclusions.

---

# Instructor Notes

This lab can be adapted in several ways:

- use Debian cloud images instead of Ubuntu,
- replace QEMU user networking with bridged networking,
- add password configuration or SSH hardening,
- integrate the lab into a broader image-building workflow.

For a shorter session, the instructor may provide prewritten `user-data` and `meta-data` files and focus on the boot methods only.

For a more advanced session, students may be asked to:

- automate the whole workflow with shell scripts,
- compare Cloud-Init with installation-time automation systems,
- experiment with more advanced modules such as `bootcmd`, `write_files`, and `packages`.
