# Orange Pi Zero 3W: Enable the TUN Kernel Module for Tailscale

This note documents a minimal, reproducible procedure for enabling TUN support on an Orange Pi Zero 3W running the vendor Ubuntu Jammy image.

The goal is not to rebuild the whole operating system image. The goal is to build and install only the missing `tun.ko` module that matches the currently running kernel.

---


## 1. Tested environment

| Item | Value |
|---|---|
| Board | Orange Pi Zero 3W |
| Memory | 4 GB |
| Storage | 64 GB TF card |
| OS | Ubuntu Jammy |
| Kernel | `6.6.98-sun60iw2` |
| Architecture | `aarch64` |
| Use case | Lightweight SwarmDock / edge robotics node |
| Target feature | Tailscale standard mode with `tailscale0` |

Check your running kernel:

```bash
uname -a
uname -r
```

Example:

```text
Linux <hostname> 6.6.98-sun60iw2 #1.0.0 SMP PREEMPT ... aarch64 GNU/Linux
```

---

## 2. Problem symptoms

After installing Tailscale, `tailscaled` may fail to start because the kernel does not provide TUN support.

Typical symptoms:

```bash
sudo modprobe tun
```

Output:

```text
modprobe: FATAL: Module tun not found in directory /lib/modules/6.6.98-sun60iw2
```

Creating `/dev/net/tun` manually is not enough:

```bash
sudo install -d -m 755 /dev/net
sudo mknod -m 666 /dev/net/tun c 10 200 2>/dev/null || true

ls -l /dev/net/tun
cat /dev/net/tun
```

If the kernel driver is missing, the result may be:

```text
crw-rw-rw- 1 root root 10, 200 /dev/net/tun
cat: /dev/net/tun: No such device
```

This means the device node exists, but the kernel has no TUN driver attached to it.

---

## 3. Solution overview

The minimal fix is:

```text
1. Install kernel build dependencies.
2. Download the matching Orange Pi linux-orangepi kernel source.
3. Reuse the current kernel configuration.
4. Enable CONFIG_TUN=m.
5. Ensure make kernelrelease matches uname -r exactly.
6. Build only drivers/net/tun.ko.
7. Install tun.ko into /lib/modules/$(uname -r)/.
8. Run depmod and modprobe tun.
9. Verify /dev/net/tun.
10. Start Tailscale.
```

The final success marker is:

```bash
cat /dev/net/tun
```

Expected output:

```text
cat: /dev/net/tun: File descriptor in bad state
```

This message is expected. It means the TUN driver is present and responding.

---

## 4. Install build dependencies

```bash
sudo apt update

sudo apt install -y \
  git make gcc g++ bc bison flex libssl-dev libelf-dev \
  dwarves pahole rsync kmod cpio xz-utils ca-certificates \
  wget curl tar
```

---

## 5. Download the Orange Pi kernel source

Create a build directory:

```bash
mkdir -p ~/kernel-build
cd ~/kernel-build
```

Option A: use `git clone`:

```bash
git clone --depth=1 \
  -b orange-pi-6.6-sun60iw2 \
  https://github.com/orangepi-xunlong/linux-orangepi.git
```

Option B: use a source tarball if `git clone` is unstable:

```bash
wget -c -O linux-orangepi-6.6-sun60iw2.tar.gz \
  https://github.com/orangepi-xunlong/linux-orangepi/archive/refs/heads/orange-pi-6.6-sun60iw2.tar.gz

tar -xzf linux-orangepi-6.6-sun60iw2.tar.gz
mv linux-orangepi-orange-pi-6.6-sun60iw2 linux-orangepi
```

Enter the source tree:

```bash
cd ~/kernel-build/linux-orangepi
```

---

## 6. Reuse the current kernel configuration

```bash
cp /boot/config-$(uname -r) .config 2>/dev/null || zcat /proc/config.gz > .config
```

Check whether TUN is already enabled:

```bash
grep CONFIG_TUN .config || true
```

Typical vendor image result:

```text
# CONFIG_TUN is not set
# CONFIG_TUN_VNET_CROSS_LE is not set
```

---

## 7. Enable TUN as a module

```bash
./scripts/config --module TUN
make olddefconfig
```

Verify:

```bash
grep CONFIG_TUN .config
```

Expected:

```text
CONFIG_TUN=m
# CONFIG_TUN_VNET_CROSS_LE is not set
```

---

## 8. Fix the kernel local version

The module must match the running kernel version exactly.

Check:

```bash
make kernelrelease
uname -r
```

If you see:

```text
6.6.98
6.6.98-sun60iw2
```

the module will not match. Fix `LOCALVERSION`:

```bash
./scripts/config --set-str LOCALVERSION "-sun60iw2"
./scripts/config --disable LOCALVERSION_AUTO
```

Remove stale generated files:

```bash
rm -f include/config/auto.conf
rm -f include/generated/utsrelease.h
rm -f include/config/kernel.release
```

Regenerate:

```bash
make olddefconfig
make prepare
```

Verify:

```bash
grep CONFIG_TUN .config
grep -E 'CONFIG_LOCALVERSION|CONFIG_LOCALVERSION_AUTO' .config
make kernelrelease
uname -r
```

Expected:

```text
CONFIG_TUN=m
CONFIG_LOCALVERSION="-sun60iw2"
# CONFIG_LOCALVERSION_AUTO is not set
6.6.98-sun60iw2
6.6.98-sun60iw2
```

---

## 9. Build `tun.ko`

Prepare the module build environment:

```bash
make -j$(nproc) prepare modules_prepare
```

Build only the TUN module:

```bash
make -j$(nproc) KBUILD_MODPOST_WARN=1 drivers/net/tun.ko
```

You may see warnings such as:

```text
WARNING: vmlinux.o is missing.
WARNING: modpost: ... undefined!
```

For this minimal module build, the warnings may be acceptable if the `.ko` is created and the `vermagic` matches the running kernel.

Check:

```bash
ls -l drivers/net/tun.ko
modinfo drivers/net/tun.ko | grep vermagic
uname -r
```

Expected:

```text
vermagic:       6.6.98-sun60iw2 SMP preempt mod_unload aarch64
6.6.98-sun60iw2
```

---

## 10. Install and load `tun.ko`

```bash
sudo install -D -m 644 drivers/net/tun.ko \
  /lib/modules/$(uname -r)/kernel/drivers/net/tun.ko

sudo depmod -a
sudo modprobe -v tun
```

Successful `modprobe` output should include:

```text
insmod /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/tun.ko
```

Check:

```bash
lsmod | grep '^tun'
```

Expected:

```text
tun                    53248  0
```

---

## 11. Verify `/dev/net/tun`

```bash
sudo install -d -m 755 /dev/net
sudo mknod -m 666 /dev/net/tun c 10 200 2>/dev/null || true

ls -l /dev/net/tun
cat /dev/net/tun
```

Expected:

```text
crw-rw-rw- 1 root root 10, 200 ... /dev/net/tun
cat: /dev/net/tun: File descriptor in bad state
```

This confirms:

```text
TUN is loaded.
The /dev/net/tun node is handled by the kernel driver.
Tailscale standard mode can create tailscale0.
```

---

## 12. Make TUN persistent across reboot

Autoload the module:

```bash
echo tun | sudo tee /etc/modules-load.d/tun.conf
```

Persistently create the device node:

```bash
echo 'c /dev/net/tun 0666 root root - 10:200' | \
  sudo tee /etc/tmpfiles.d/tun.conf

sudo systemd-tmpfiles --create /etc/tmpfiles.d/tun.conf
```

Reboot and verify:

```bash
sudo reboot
```

After reconnecting:

```bash
lsmod | grep '^tun'
ls -l /dev/net/tun
cat /dev/net/tun
```

Expected:

```text
tun                    53248  0
cat: /dev/net/tun: File descriptor in bad state
```

---

## 13. Start Tailscale

If Tailscale is already installed:

```bash
sudo systemctl enable --now tailscaled
sudo systemctl restart tailscaled
sleep 3
sudo tailscale up
```

If Tailscale is not installed:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Check:

```bash
tailscale ip -4
tailscale status
ip addr show tailscale0
```

Expected:

```text
tailscale0 exists
the node has a 100.x.x.x Tailscale address
tailscale status shows the node online
```

---

## 14. Final validation checklist

```bash
uname -r
make -C ~/kernel-build/linux-orangepi kernelrelease

lsmod | grep '^tun'
cat /dev/net/tun

tailscale ip -4
tailscale status
ip addr show tailscale0
```

Expected final state:

```text
Kernel release matches uname -r.
tun module is loaded.
cat /dev/net/tun returns "File descriptor in bad state".
tailscale0 exists.
The node is online in Tailscale.
```

---

## 15. Notes and maintenance

### 15.1 Do not upgrade the kernel casually

The module built here is specific to:

```text
6.6.98-sun60iw2
```

If the kernel changes, rebuild `tun.ko` for the new kernel:

```bash
uname -r
sudo modprobe tun
cat /dev/net/tun
```

### 15.2 Back up the module

```bash
mkdir -p ~/swarmdock/backups/kernel-modules

cp /lib/modules/$(uname -r)/kernel/drivers/net/tun.ko \
  ~/swarmdock/backups/kernel-modules/tun.ko-$(uname -r)
```

### 15.3 Useful paths

```text
Kernel source:
~/kernel-build/linux-orangepi

Built module:
~/kernel-build/linux-orangepi/drivers/net/tun.ko

Installed module:
 /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/tun.ko
```

---

## 16. Relevance to SwarmDock

After enabling TUN, the Orange Pi Zero 3W can be used as a more complete lightweight SwarmDock edge node:

```text
wlan0       LAN access
tailscale0  remote management overlay
serial      LoRa / radio / sensor integration
I2C         peripheral and sensor integration
Python/C++  lightweight node software
```

Suggested node identity:

```bash
export SP_HOST_ID=sp-orange-zero3w
export SP_NODE_ROLE=edge-lite
export SP_NODE_BOARD=orange-pi-zero-3w-4g
```

Positioning:

```text
Orange Pi Zero 3W = lightweight SwarmDock edge node
                  + Tailscale remote node
                  + serial / I2C integration node
```
