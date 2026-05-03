# mt76 Jumbo Frame / Large MTU Support

This branch raises the maximum MTU ceiling for all mt76-supported adapters,
allowing commands such as:

```bash
sudo ip link set wlan0 mtu 4000
```

to succeed and work correctly instead of being rejected by the kernel.

---

## Background

The standard 802.11 data frame payload limit is 2304 bytes, which maps to a
network MTU of 2272 bytes after 802.11 header overhead. The Linux kernel's
default for mac80211-based WiFi interfaces is 1500 bytes (standard Ethernet
MTU), but the ceiling enforced by the driver typically matches this 2304-byte
limit.

Frames larger than this limit — "jumbo frames" — are useful in environments
where:

- The WiFi link is bridged to a wired network that uses jumbo frames (e.g. a
  NAS or storage server with 9000-byte MTU)
- High-throughput, low-CPU-overhead transfers are needed on a local network
  where you control both endpoints
- You are running a tunnel or VPN inside the WiFi link and need headroom for
  encapsulation headers without fragmenting at the IP layer

---

## What Changed

**File:** `mac80211.c`, inside `mt76_phy_init()` — one line added:

```c
hw->max_mtu = 7500;
```

### Why this works

mt76 is a mac80211-based driver. It does not create network interfaces directly
— that is done by the mac80211 kernel layer. When mac80211 creates a virtual
interface (station, AP, etc.) it reads `hw->max_mtu` and sets
`netdev->max_mtu` on the underlying network device to that value.

`netdev->max_mtu` is the value that `ip link set mtu` is checked against before
the kernel even calls any driver callback. Without this change, the kernel
silently rejects any MTU above the default ceiling before the driver is
consulted.

Setting `hw->max_mtu = 7500` in `mt76_phy_init()` — which runs before
`ieee80211_register_hw()` — ensures the value is in place for every interface
mac80211 subsequently creates.

### Why no TX path changes are needed

mac80211 handles 802.11-layer fragmentation internally. Frames larger than the
802.11 MPDU limit are automatically split into multiple MPDUs by the mac80211 TX
path before they are handed to the driver for transmission. The driver's DMA and
queue code never sees a single oversized frame.

### Why 7500 bytes

The 802.11 HT A-MSDU hard limit is 7935 bytes. Staying at 7500 bytes leaves
435 bytes of headroom for 802.11, IP, and TCP/UDP headers, ensuring that a
7500-byte MTU does not produce A-MSDUs that exceed the hardware limit. This
accommodates the common 4096-byte and 4352-byte large-MTU configurations with
comfortable margin.

---

## Limitations

- **Both endpoints must support the MTU.** Setting `mtu 4000` on your WiFi
  interface only controls the local transmit path. The remote peer (AP or
  another station) must also have a matching or larger MTU, or frames will be
  fragmented at the IP layer regardless.

- **The AP must support large A-MSDUs.** Most modern APs do, but older or
  embedded hardware may not. If connectivity degrades after raising the MTU,
  lower it and test with a smaller value.

- **Performance may vary.** Jumbo frames reduce per-packet overhead and can
  improve throughput for large transfers. They may increase latency for small
  packets (e.g. interactive traffic) due to head-of-line blocking. Use jumbo
  frames for dedicated bulk-transfer links, not mixed workloads.

- **This change is independent of the connectivity optimisation branch.**
  It can be applied alone or combined with those changes.

---

## Building

### Prerequisites

You need the kernel headers for your running kernel and standard build tools.

**Debian / Ubuntu / Raspberry Pi OS:**
```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r)
```

**Fedora / RHEL / CentOS:**
```bash
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

**Arch Linux:**
```bash
sudo pacman -S linux-headers
```

### Compile

From the root of this repository:

```bash
make -C /lib/modules/$(uname -r)/build M=$PWD
```

A successful build produces `.ko` files in each chip subdirectory and in the
root directory. Only `mt76.ko` (the core module) was changed, but all modules
must be rebuilt together against the same source tree to ensure consistency.

---

## Loading the Custom Driver

### Identify your chip and current module

```bash
# For PCIe adapters
lspci -k | grep -A3 "Network\|Wireless"

# For USB adapters
lsusb
# Then:
lsmod | grep mt76
```

Common chip-to-module mappings:

| Chip | Module |
|---|---|
| MT7603E | `mt7603/mt7603e.ko` |
| MT7612E / MT7602E | `mt76x2/mt76x2e.ko` |
| MT7612U | `mt76x2/mt76x2u.ko` |
| MT7610U | `mt76x0/mt76x0u.ko` |
| MT7615E | `mt7615/mt7615e.ko` |
| MT7663U | `mt7615/mt7663u.ko` |
| MT7663S | `mt7615/mt7663s.ko` |
| MT7915E | `mt7915/mt7915e.ko` |
| MT7921E | `mt7921/mt7921e.ko` |
| MT7921U | `mt7921/mt7921u.ko` |
| MT7921S | `mt7921/mt7921s.ko` |
| MT7925E | `mt7925/mt7925e.ko` |
| MT7925U | `mt7925/mt7925u.ko` |
| MT7996E | `mt7996/mt7996e.ko` |

### Unload the mainline driver

```bash
# Bring the interface down first if connected
sudo ip link set wlan0 down

# Unload your chip module (example: MT7921 PCIe)
sudo modprobe -r mt7921e
# Core modules unload automatically once nothing depends on them
```

### Load the custom modules

The core `mt76.ko` must be loaded first. Load the connac and mt792x libraries
if your chip needs them (MT7615, MT7921, MT7925, MT7996), then the chip module.

```bash
# Core — always required
sudo insmod mt76.ko

# Connac library — required for MT7615, MT7921, MT7925, MT7996
sudo insmod mt76-connac-lib.ko

# MT792x library — required for MT7921, MT7925
sudo insmod mt792x-lib.ko

# Your chip module (example: MT7921 PCIe)
sudo insmod mt7921/mt7921-common.ko
sudo insmod mt7921/mt7921e.ko
```

For USB chips, also load the USB bridge:
```bash
sudo insmod mt76-usb.ko
sudo insmod mt792x-usb.ko      # MT792x USB only
sudo insmod mt7921/mt7921u.ko  # example: MT7921 USB
```

### Verify the new MTU ceiling

Once the interface is up, confirm the raised ceiling is in effect:

```bash
ip link show wlan0
# You should see: maxmtu 7500

# Now raise the MTU:
sudo ip link set wlan0 mtu 4000

# Confirm it was accepted:
ip link show wlan0 | grep mtu
# Should show: mtu 4000
```

Test with a large ping to ensure the AP and network path accept the larger
frames:

```bash
# -M do: do not fragment   -s 3900: payload size
ping -M do -s 3900 192.168.1.1
```

If you see `Message too long` or packet loss, the AP or network path is
rejecting the larger frames. Try a smaller MTU (e.g. 2000 or 2500) and test
again.

---

## Making the Change Persistent Across Reboots

### Step 1 — Install the modules

```bash
KVER=$(uname -r)
sudo mkdir -p /lib/modules/$KVER/updates/mt76

# Copy all built modules
find . -name "*.ko" | sudo xargs -I{} cp {} /lib/modules/$KVER/updates/mt76/

# Rebuild module dependency database
sudo depmod -a
```

Modules in `updates/` are preferred over the kernel's built-in versions
automatically.

### Step 2 — Set the MTU at boot

The MTU is a per-interface setting that resets on each load. Persist it using
one of these methods:

**Using `/etc/network/interfaces` (Debian/Ubuntu):**
```
iface wlan0 inet dhcp
    pre-up ip link set wlan0 mtu 4000
```

**Using NetworkManager (most modern distros):**
```bash
# Replace <connection-name> with the output of: nmcli connection show
nmcli connection modify <connection-name> 802-11-wireless.mtu 4000
```

**Using a systemd service (universal):**

Create `/etc/systemd/system/wlan-mtu.service`:
```ini
[Unit]
Description=Set WiFi jumbo frame MTU
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set wlan0 mtu 4000
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Then enable it:
```bash
sudo systemctl enable wlan-mtu.service
```

---

## Reverting to the Mainline Driver

Remove the installed modules and rebuild the module index:

```bash
KVER=$(uname -r)
sudo rm -rf /lib/modules/$KVER/updates/mt76
sudo depmod -a
sudo modprobe -r mt7921e    # your chip module
sudo modprobe mt7921e       # reloads the mainline version
```

After reverting, `ip link set wlan0 mtu 4000` will fail again with
`RTNETLINK answers: Message too long`, which confirms the mainline ceiling is
back in effect.
