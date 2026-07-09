# Orange Pi Zero 3W: AR9271 USB Wi-Fi with IBSS + BATMAN-ADV Mesh

This note documents a reproducible engineering procedure for using an AR9271-based USB Wi-Fi adapter as an IBSS/ad-hoc underlay for BATMAN-ADV on an Orange Pi Zero 3W.

The final result is:

```text
wlan0              normal Wi-Fi management network
tailscale0         remote management overlay
<usb-wifi-iface>   AR9271 USB Wi-Fi in IBSS/ad-hoc mode
bat0               BATMAN-ADV Layer-2 mesh interface
```

This procedure does not require rebuilding the entire operating system image. It builds and installs only the missing kernel modules required by the current vendor kernel.

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
| USB Wi-Fi chipset | Atheros AR9271 |
| USB ID seen before firmware load | `040d:3801` |
| USB ID seen after re-enumeration | `0cf3:9271` |
| Driver | `ath9k_htc` |
| Mesh mode used | IBSS/ad-hoc + BATMAN-ADV |
| 802.11s mesh point | Not supported by this adapter in this setup |

Replace the interface name and IP addresses in this note with your own values.

Example values used in this note:

```text
USB Wi-Fi interface: wlx00127b2174f3
IBSS SSID:           SwarmNet
Frequency:           2412 MHz
Fixed BSSID:         02:12:34:56:78:9a
bat0 IP:             10.10.10.35/24
```

---

## 2. Target network layout

Keep the board's built-in Wi-Fi as the management network. Do not convert it to mesh mode.

```text
wlan0
  -> normal Wi-Fi
  -> SSH / package install / LAN access
  -> Tailscale underlay

tailscale0
  -> remote maintenance

wlx00127b2174f3
  -> AR9271 USB Wi-Fi
  -> IBSS/ad-hoc underlay

bat0
  -> BATMAN-ADV virtual Layer-2 mesh interface
  -> example IP: 10.10.10.35/24
```

---

## 3. Initial problem symptoms

After plugging in the AR9271 USB Wi-Fi adapter, `lsusb` can see the device:

```bash
lsusb
```

Example:

```text
ID 040d:3801 VIA Technologies, Inc. VIA USB2.0 WLAN
```

But `iw dev` may only show the built-in Wi-Fi interface:

```text
Interface wlan0
```

No `wlan1` or `wlx...` interface appears.

The USB topology may show that no driver is bound:

```bash
lsusb -t
```

Example:

```text
Class=Vendor Specific Class, Driver=, 480M
```

Trying to load the expected driver may fail:

```bash
sudo modprobe -v ath9k_htc
```

Example:

```text
modprobe: FATAL: Module ath9k_htc not found in directory /lib/modules/6.6.98-sun60iw2
```

This means the vendor kernel does not provide the required `ath9k_htc` module.

---

## 4. Solution overview

The reproducible fix is:

```text
1. Install firmware and Wi-Fi tools.
2. Build and install the missing ath9k_htc driver modules.
3. Patch ath9k_htc to avoid unresolved mac80211 LED symbols if required.
4. Load ath9k_htc and confirm that the USB Wi-Fi interface appears.
5. Build and load batman-adv if the vendor kernel does not provide it.
6. Use IBSS/ad-hoc mode because this adapter does not expose mesh point mode.
7. Add the IBSS interface to BATMAN-ADV.
8. Bring up bat0 and assign a mesh IP.
```

Important distinction:

```text
Not used:  802.11s mesh point + BATMAN-ADV
Used:      IBSS/ad-hoc + BATMAN-ADV
```

---

## 5. Install tools and firmware

```bash
sudo apt update

sudo apt install -y \
  batctl \
  iw \
  wireless-tools \
  bridge-utils \
  iproute2 \
  kmod \
  rfkill \
  usbutils \
  linux-firmware
```

Check firmware files:

```bash
ls -l /lib/firmware/htc_9271.fw /lib/firmware/htc_7010.fw 2>/dev/null
```

The driver may also request this firmware path at runtime:

```text
ath9k_htc/htc_9271-1.4.0.fw
```

Installing `linux-firmware` normally provides the required files.

---

## 6. Prepare the Orange Pi kernel source

Create a build directory:

```bash
mkdir -p ~/kernel-build
cd ~/kernel-build
```

Download the Orange Pi vendor kernel source:

```bash
wget -c -O linux-orangepi-6.6-sun60iw2.tar.gz \
  https://github.com/orangepi-xunlong/linux-orangepi/archive/refs/heads/orange-pi-6.6-sun60iw2.tar.gz

tar -xzf linux-orangepi-6.6-sun60iw2.tar.gz
mv linux-orangepi-orange-pi-6.6-sun60iw2 linux-orangepi
cd linux-orangepi
```

Reuse the current kernel configuration:

```bash
cp /boot/config-$(uname -r) .config 2>/dev/null || zcat /proc/config.gz > .config
```

---

## 7. Ensure the kernel release matches

The module version must match the running kernel exactly.

```bash
make kernelrelease
uname -r
```

If needed, set the local version:

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
make kernelrelease
uname -r
```

Expected:

```text
6.6.98-sun60iw2
6.6.98-sun60iw2
```

---

## 8. Enable AR9271 / ath9k_htc modules

```bash
./scripts/config --enable WLAN
./scripts/config --enable WLAN_VENDOR_ATH

./scripts/config --module ATH
./scripts/config --module ATH9K_HW
./scripts/config --module ATH9K_COMMON
./scripts/config --module ATH9K_HTC

./scripts/config --disable ATH9K_HTC_DEBUGFS
```

Regenerate configuration:

```bash
rm -f include/config/auto.conf
rm -f include/generated/utsrelease.h
rm -f include/config/kernel.release

make olddefconfig
make prepare
```

Check:

```bash
grep -E 'CONFIG_WLAN|CONFIG_WLAN_VENDOR_ATH|CONFIG_ATH|CONFIG_ATH9K' .config
```

Expected values include:

```text
CONFIG_WLAN=y
CONFIG_WLAN_VENDOR_ATH=y
CONFIG_ATH_COMMON=m
CONFIG_ATH9K_HW=m
CONFIG_ATH9K_COMMON=m
CONFIG_ATH9K_HTC=m
```

---

## 9. Build the ath9k_htc-related modules

```bash
make -j$(nproc) prepare modules_prepare

make -j$(nproc) KBUILD_MODPOST_WARN=1 \
  drivers/net/wireless/ath/ath.ko \
  drivers/net/wireless/ath/ath9k/ath9k_hw.ko \
  drivers/net/wireless/ath/ath9k/ath9k_common.ko \
  drivers/net/wireless/ath/ath9k/ath9k_htc.ko
```

Check:

```bash
ls -l drivers/net/wireless/ath/ath.ko
ls -l drivers/net/wireless/ath/ath9k/ath9k_hw.ko
ls -l drivers/net/wireless/ath/ath9k/ath9k_common.ko
ls -l drivers/net/wireless/ath/ath9k/ath9k_htc.ko

modinfo drivers/net/wireless/ath/ath9k/ath9k_htc.ko | grep -E 'vermagic|depends'
uname -r
```

Expected:

```text
depends:        ath9k_hw,ath9k_common,ath
vermagic:       6.6.98-sun60iw2 SMP preempt mod_unload aarch64
6.6.98-sun60iw2
```

---

## 10. Install the ath9k_htc-related modules

Using a script reduces command-copying mistakes:

```bash
cat > /tmp/install-ath9k-htc.sh <<'EOF'
#!/usr/bin/env bash
set -e

KVER="$(uname -r)"
SRC="$HOME/kernel-build/linux-orangepi"
DST="/lib/modules/$KVER/kernel/drivers/net/wireless/ath"

echo "Kernel: $KVER"
echo "Source: $SRC"
echo "Dest  : $DST"

sudo install -D -m 644 "$SRC/drivers/net/wireless/ath/ath.ko" \
  "$DST/ath.ko"

sudo install -D -m 644 "$SRC/drivers/net/wireless/ath/ath9k/ath9k_hw.ko" \
  "$DST/ath9k/ath9k_hw.ko"

sudo install -D -m 644 "$SRC/drivers/net/wireless/ath/ath9k/ath9k_common.ko" \
  "$DST/ath9k/ath9k_common.ko"

sudo install -D -m 644 "$SRC/drivers/net/wireless/ath/ath9k/ath9k_htc.ko" \
  "$DST/ath9k/ath9k_htc.ko"

sudo depmod -a "$KVER"

echo "===== installed files ====="
find "/lib/modules/$KVER" -name 'ath*.ko' | sort

echo "===== modinfo ====="
modinfo ath9k_htc | grep -E 'filename|depends|vermagic'
EOF

bash /tmp/install-ath9k-htc.sh
```

---

## 11. Handle unresolved mac80211 LED symbols

On this vendor kernel, a first attempt to load `ath9k_htc` may fail:

```bash
sudo modprobe -v ath9k_htc
```

Kernel log:

```bash
sudo dmesg | grep -iE 'ath|htc|firmware|wlan|wifi|usb' | tail -120
```

Possible error:

```text
ath9k_htc: Unknown symbol __ieee80211_create_tpt_led_trigger (err -2)
ath9k_htc: Unknown symbol __ieee80211_get_radio_led_name (err -2)
```

This means `ath9k_htc.ko` references mac80211 LED helper symbols that are not available from the running kernel.

Check:

```bash
nm -u drivers/net/wireless/ath/ath9k/ath9k_htc.ko \
  | grep -E 'ieee80211.*led|tpt_led|radio_led' \
  || echo "no led symbol dependency"
```

If unresolved LED symbols appear, apply the no-LED patch below.

---

## 12. Apply a minimal no-LED patch to ath9k_htc

Search for the LED-related code:

```bash
grep -R -n "CONFIG_MAC80211_LEDS\|ieee80211_create_tpt_led_trigger\|ieee80211_get_radio_led_name" \
  drivers/net/wireless/ath/ath9k
```

Back up the ath9k directory:

```bash
cp -a drivers/net/wireless/ath/ath9k \
  drivers/net/wireless/ath/ath9k.bak.$(date +%Y%m%d_%H%M%S)
```

Disable only the HTC driver LED code paths:

```bash
grep -R -l "CONFIG_MAC80211_LEDS" drivers/net/wireless/ath/ath9k/htc* \
  | xargs sed -i 's/CONFIG_MAC80211_LEDS/CONFIG_MAC80211_LEDS_DISABLED_FOR_ORANGEPI/g'
```

Verify:

```bash
grep -R -n "CONFIG_MAC80211_LEDS\|CONFIG_MAC80211_LEDS_DISABLED_FOR_ORANGEPI" \
  drivers/net/wireless/ath/ath9k/htc*
```

Expected:

```text
#ifdef CONFIG_MAC80211_LEDS_DISABLED_FOR_ORANGEPI
```

Because this macro is intentionally undefined, the LED-related code will no longer be compiled into `ath9k_htc.ko`.

---

## 13. Rebuild and install the no-LED ath9k_htc module

```bash
cat > /tmp/rebuild-ath9k-htc-noled.sh <<'EOF'
#!/usr/bin/env bash
set -e

cd "$HOME/kernel-build/linux-orangepi"

echo "===== clean ath modules ====="
make M=drivers/net/wireless/ath clean

echo "===== prepare ====="
make -j$(nproc) prepare modules_prepare

echo "===== build ath9k_htc no-led ====="
make -j$(nproc) KBUILD_MODPOST_WARN=1 \
  drivers/net/wireless/ath/ath.ko \
  drivers/net/wireless/ath/ath9k/ath9k_hw.ko \
  drivers/net/wireless/ath/ath9k/ath9k_common.ko \
  drivers/net/wireless/ath/ath9k/ath9k_htc.ko

echo "===== check led symbol dependency ====="
nm -u drivers/net/wireless/ath/ath9k/ath9k_htc.ko \
  | grep -E 'ieee80211.*led|tpt_led|radio_led' \
  || echo "no led symbol dependency"

echo "===== modinfo ====="
modinfo drivers/net/wireless/ath/ath9k/ath9k_htc.ko | grep -E 'vermagic|depends'
uname -r

echo "===== install modules ====="
KVER="$(uname -r)"
DST="/lib/modules/$KVER/kernel/drivers/net/wireless/ath"

sudo install -D -m 644 drivers/net/wireless/ath/ath.ko \
  "$DST/ath.ko"

sudo install -D -m 644 drivers/net/wireless/ath/ath9k/ath9k_hw.ko \
  "$DST/ath9k/ath9k_hw.ko"

sudo install -D -m 644 drivers/net/wireless/ath/ath9k/ath9k_common.ko \
  "$DST/ath9k/ath9k_common.ko"

sudo install -D -m 644 drivers/net/wireless/ath/ath9k/ath9k_htc.ko \
  "$DST/ath9k/ath9k_htc.ko"

sudo depmod -a

echo "===== modinfo installed ====="
modinfo ath9k_htc | grep -E 'filename|depends|vermagic'
EOF

bash /tmp/rebuild-ath9k-htc-noled.sh
```

Success marker:

```text
no led symbol dependency
```

Installed module example:

```text
filename:       /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/wireless/ath/ath9k/ath9k_htc.ko
depends:        ath9k_hw,ath,ath9k_common
vermagic:       6.6.98-sun60iw2 SMP preempt mod_unload aarch64
```

---

## 14. Load ath9k_htc

```bash
sudo modprobe -r ath9k_htc ath9k_common ath9k_hw ath 2>/dev/null || true
sudo modprobe -v ath9k_htc
```

Successful output:

```text
insmod /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/wireless/ath/ath.ko
insmod /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/wireless/ath/ath9k/ath9k_hw.ko
insmod /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/wireless/ath/ath9k/ath9k_common.ko
insmod /lib/modules/6.6.98-sun60iw2/kernel/drivers/net/wireless/ath/ath9k/ath9k_htc.ko
```

Check:

```bash
lsmod | grep -E 'ath9k|ath|mac80211|cfg80211'
```

Expected:

```text
ath9k_htc
ath9k_common
ath9k_hw
ath
```

---

## 15. Confirm the USB Wi-Fi interface

Unplug and replug the USB Wi-Fi adapter, then check:

```bash
sudo dmesg | grep -iE '040d|3801|0cf3|9271|ath|htc|firmware|wlan|wifi|usb' | tail -120
```

Successful log examples:

```text
ath9k_htc: Firmware ath9k_htc/htc_9271-1.4.0.fw requested
ath9k_htc: Transferred FW: ath9k_htc/htc_9271-1.4.0.fw
ath9k_htc: FW Version: 1.4
ieee80211 phy1: Atheros AR9271 Rev:1
ath9k_htc ... wlx00127b2174f3: renamed from wlan1
```

Check:

```bash
iw dev
ip link
```

Expected:

```text
Interface wlx00127b2174f3
type managed
```

The actual interface name may differ. Replace `wlx00127b2174f3` in the rest of this note with your own interface name.

---

## 16. Confirm supported Wi-Fi modes

```bash
iw phy phy1 info | grep -A12 "Supported interface modes"
```

Example for this adapter:

```text
Supported interface modes:
     * IBSS
     * managed
     * AP
     * AP/VLAN
     * monitor
     * P2P-client
     * P2P-GO
     * outside context of a BSS
```

There is no:

```text
* mesh point
```

Therefore:

```text
802.11s mesh point is not available in this setup.
IBSS/ad-hoc is available.
BATMAN-ADV can run on top of the IBSS interface.
```

---

## 17. Build or load BATMAN-ADV

Install `batctl`:

```bash
sudo apt install -y batctl
```

Try loading the kernel module:

```bash
sudo modprobe -v batman-adv
batctl -v
```

If the module is missing, build it from the same kernel source:

```bash
cd ~/kernel-build/linux-orangepi

./scripts/config --module BATMAN_ADV
./scripts/config --enable BATMAN_ADV_BATMAN_V
./scripts/config --enable BATMAN_ADV_BLA
./scripts/config --enable BATMAN_ADV_DAT
./scripts/config --enable BATMAN_ADV_NC
./scripts/config --disable BATMAN_ADV_DEBUG

make olddefconfig
make prepare
make -j$(nproc) prepare modules_prepare

make -j$(nproc) KBUILD_MODPOST_WARN=1 net/batman-adv/batman-adv.ko
```

Install and load:

```bash
sudo install -D -m 644 net/batman-adv/batman-adv.ko \
  /lib/modules/$(uname -r)/kernel/net/batman-adv/batman-adv.ko

sudo depmod -a
sudo modprobe -v batman-adv
```

Make it load on boot:

```bash
echo batman-adv | sudo tee /etc/modules-load.d/batman-adv.conf
```

Verify:

```bash
lsmod | grep batman
batctl -v
```

Expected:

```text
batman_adv
batctl ... [batman-adv: ...]
```

---

## 18. Configure IBSS + BATMAN-ADV

The following script configures the USB Wi-Fi adapter in IBSS/ad-hoc mode and attaches it to BATMAN-ADV.

Edit these variables before use:

```text
IFACE   = your AR9271 interface name
MESH_ID = IBSS network name
FREQ    = Wi-Fi frequency in MHz
BSSID   = fixed IBSS BSSID shared by all nodes
BAT_IP  = unique bat0 IP for this node
```

Example script:

```bash
cat > /tmp/setup-batman-ibss.sh <<'EOF'
#!/usr/bin/env bash
set -e

IFACE="wlx00127b2174f3"
MESH_ID="SwarmNet"
FREQ="2412"
BSSID="02:12:34:56:78:9a"
BAT_IP="10.10.10.35/24"

echo "===== load modules ====="
sudo modprobe batman-adv
sudo modprobe ath9k_htc

echo "===== keep NetworkManager away from USB mesh iface ====="
sudo nmcli dev set "$IFACE" managed no 2>/dev/null || true

echo "===== clean old batman state ====="
sudo batctl if del "$IFACE" 2>/dev/null || true
sudo ip addr flush dev bat0 2>/dev/null || true
sudo ip link set bat0 down 2>/dev/null || true

echo "===== configure USB Wi-Fi as IBSS/ad-hoc ====="
sudo ip link set "$IFACE" down
sudo ip addr flush dev "$IFACE"
sudo iw dev "$IFACE" set type ibss
sudo ip link set "$IFACE" up

echo "===== join IBSS network ====="
sudo iw dev "$IFACE" ibss join "$MESH_ID" "$FREQ" HT20 fixed-freq "$BSSID"

sleep 2

echo "===== add USB Wi-Fi to BATMAN-ADV ====="
sudo batctl if add "$IFACE"
sudo ip link set up dev bat0
sudo ip addr add "$BAT_IP" dev bat0

echo "===== status ====="
iw dev
iw dev "$IFACE" link || true
batctl if
ip addr show bat0
EOF

bash /tmp/setup-batman-ibss.sh
```

---

## 19. Verify IBSS + BATMAN-ADV

Check the USB Wi-Fi mode:

```bash
iw dev
```

Expected:

```text
Interface wlx00127b2174f3
ssid SwarmNet
type IBSS
channel 1 (2412 MHz), width: 20 MHz
```

Check the IBSS link:

```bash
iw dev wlx00127b2174f3 link
```

Expected:

```text
Joined IBSS 02:12:34:56:78:9a (on wlx00127b2174f3)
SSID: SwarmNet
freq: 2412
```

Check BATMAN-ADV:

```bash
batctl if
ip addr show bat0
```

Expected:

```text
wlx00127b2174f3: active

bat0: <BROADCAST,MULTICAST,UP,LOWER_UP>
inet 10.10.10.35/24 scope global bat0
```

At this point, the local mesh interface is ready.

---

## 20. Configure additional mesh nodes

A single node will not show neighbors.

On every additional node, use the same IBSS parameters:

```text
MESH_ID = SwarmNet
FREQ    = 2412
BSSID   = 02:12:34:56:78:9a
```

Use a unique `bat0` IP per node:

```text
Node A: 10.10.10.35/24
Node B: 10.10.10.3/24
Node C: 10.10.10.11/24
```

Then test:

```bash
ping 10.10.10.3
batctl n
batctl o
```

If there is only one node, empty `batctl n` or `batctl o` output is normal.

---

## 21. Make the configuration persistent

### 21.1 Keep NetworkManager away from the mesh interface

```bash
sudo mkdir -p /etc/NetworkManager/conf.d

cat <<'EOF' | sudo tee /etc/NetworkManager/conf.d/10-ignore-swarmnet.conf
[keyfile]
unmanaged-devices=interface-name:wlx00127b2174f3
EOF

sudo systemctl restart NetworkManager
```

Update the interface name if your device uses a different `wlx...` name.

### 21.2 Store the setup script

```bash
mkdir -p ~/swarmdock/scripts

cp /tmp/setup-batman-ibss.sh ~/swarmdock/scripts/setup-batman-ibss.sh
chmod +x ~/swarmdock/scripts/setup-batman-ibss.sh
```

Run when needed:

```bash
~/swarmdock/scripts/setup-batman-ibss.sh
```

### 21.3 Optional module autoload

```bash
echo ath9k_htc | sudo tee /etc/modules-load.d/ath9k_htc.conf
echo batman-adv | sudo tee /etc/modules-load.d/batman-adv.conf
```

---

## 22. Final validation checklist

```bash
uname -r
modinfo ath9k_htc | grep -E 'filename|depends|vermagic'
lsmod | grep -E 'ath9k|ath'

batctl -v
lsmod | grep batman

iw dev
iw dev wlx00127b2174f3 link
batctl if
ip addr show bat0
```

Expected final state:

```text
ath9k_htc loads successfully.
The AR9271 USB Wi-Fi appears as a wlx... interface.
The interface is joined to an IBSS network.
batctl if reports the interface as active.
bat0 is UP and has a unique mesh IP.
```

---

## 23. Notes and maintenance

### 23.1 Do not upgrade the kernel casually

The locally built modules are specific to:

```text
6.6.98-sun60iw2
```

If the kernel changes, rebuild:

```text
tun.ko
batman-adv.ko
ath.ko
ath9k_hw.ko
ath9k_common.ko
ath9k_htc.ko
```

### 23.2 Do not use the management Wi-Fi as the mesh interface

Keep `wlan0` for:

```text
LAN SSH
package installation
Tailscale underlay
```

Use the USB Wi-Fi adapter for:

```text
IBSS/ad-hoc
BATMAN-ADV
bat0
```

### 23.3 Interface names can change

The interface name may look like:

```text
wlx00127b2174f3
```

If you change the USB adapter, the name will change. Check with:

```bash
iw dev
ip link
```

Update scripts accordingly.

### 23.4 IBSS is not 802.11s mesh point

This adapter does not expose:

```text
mesh point
```

Do not use:

```bash
sudo iw dev <iface> set type mesh
sudo iw dev <iface> mesh join SwarmNet freq 2412
```

Use:

```bash
sudo iw dev <iface> set type ibss
sudo iw dev <iface> ibss join SwarmNet 2412 HT20 fixed-freq 02:12:34:56:78:9a
```

### 23.5 Empty BATMAN-ADV neighbor tables are normal with one node

With only one mesh node:

```bash
batctl n
batctl o
```

may show no neighbors. Add a second node with the same IBSS parameters to test real connectivity.

---

## 24. Relevance to SwarmDock

After this procedure, the Orange Pi Zero 3W has a multi-network edge-node setup:

```text
wlan0              LAN / SSH / normal network
tailscale0         remote management overlay
AR9271 USB Wi-Fi   IBSS/ad-hoc mesh underlay
bat0               BATMAN-ADV Layer-2 mesh overlay
serial             LoRa / radio / sensor integration
I2C                peripheral and sensor integration
Python/C++         lightweight node software
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
                  + IBSS + BATMAN-ADV mesh node
                  + serial / I2C integration node
```
