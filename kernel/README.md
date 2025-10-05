# Hands-on: Compiling a Linux Kernel in a VirtualBox VM with a Separate Build Disk

This lab walks you through compiling a Linux kernel inside a VirtualBox VM, using a **second virtual disk** to store all build artifacts.  
This keeps the system clean and avoids filling up your main root disk.

---

## 1. Prepare the Virtual Machine
1. Create a new VM (e.g., Debian 13, 2–4 vCPUs, 4–8 GB RAM).
2. Add a second disk (VDI, dynamically allocated, 30 GB):
   - VM ▸ **Settings** ▸ **Storage** ▸ Controller SATA ▸ **Add Hard Disk…** (50 GB)
3. Boot the VM, install the OS, and update packages:
   ```bash
   sudo apt update && sudo apt -y upgrade
   sudo apt -y install build-essential git
   ```

---

## 2. Prepare the Extra Disk
Check the new disk (likely `/dev/sdb`):
```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

Partition, format, and mount:
```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart build ext4 1MiB 100%
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
sudo mkfs.ext4 -L KBUILD /dev/sdb1

sudo mkdir -p /build
sudo blkid /dev/sdb1   # copy the UUID
echo 'UUID=<PASTE-UUID>  /build  ext4  defaults,noatime  0  2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo mount
sudo chown -R $USER:$USER /build
```

---

## 3. Install Build Prerequisites

```bash
sudo apt -y install \
  bc bison flex libelf-dev libssl-dev libncurses-dev dwarves \
  fakeroot rsync ccache build-essential
```

Enable `ccache`:
```bash
echo 'export PATH="/usr/lib/ccache:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 4. Fetch Kernel Sources

```bash
mkdir -p ~/src && cd ~/src
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.10.tar.xz
tar -xf linux-6.10.tar.xz
cd linux-6.10
```
---

## 5. Configure the Kernel

Or start fresh:
```bash
make O=/build/linux-build menuconfig
```

Check the following parameters:
- Go to "Cryptographic API" / "Certificates for signature checking" and set "Additional X.509 keys for default system keyring" to blank.
- Verify that "File name or PKCS#11 URI of module signing key" is also set to blank.
- Disable "Enable loadable module support" / "Automatically sign all modules".
- Disable "Security Options" / "Digital signature verification using multiple keyrings".

---

## 6. Build on the Separate Disk
Set up out-of-tree build:
```bash
mkdir -p /build/linux-build /build/tmp
export TMPDIR=/build/tmp
make O=/build/linux-build -j"$(nproc)"
```

Notes:
- `O=...` keeps build artifacts out of the source tree.
- `TMPDIR` ensures temporary files go on the extra disk.

---

## 7. Install Kernel + Modules
```bash
sudo make O=/build/linux-build modules_install
sudo make O=/build/linux-build install
```

Update bootloader (if needed):
```bash
sudo update-initramfs -c -k "$(make O=/build/linux-build kernelrelease)"
sudo update-grub
```

---

## 8. Reboot and Verify
```bash
uname -r
```
You should see your new kernel version.

---

## 9. Optional: Package as .deb
Instead of manual installation, produce Debian packages:
```bash
fakeroot make -j"$(nproc)" deb-pkg
sudo dpkg -i ../linux-image-*.deb ../linux-headers-*.deb
sudo update-grub
```

This makes uninstallation easier.

---

## 10. Cleanup / Repeatability
Reset the build directory:
```bash
rm -rf /build/linux-build /build/tmp
mkdir -p /build/linux-build /build/tmp
```
