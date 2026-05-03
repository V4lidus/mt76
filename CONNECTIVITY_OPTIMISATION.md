# mt76 Connectivity Optimisation

This branch tunes the mt76 driver to prioritise connection reliability over
throughput. The principle is: a slow connection is acceptable; a dropped
connection is not. Every change trades some peak bandwidth for a measurable
reduction in spurious disconnections, failed associations, or unnecessary rate
degradation.

This is an explicit shift away from the driver's default behaviour, which is
tuned for throughput in clean RF environments. These changes are intended for
use in congested spectrum, at range, or anywhere marginal signal or interference
causes frequent disconnections.

---

## What Changed and Why

### 1. Short Guard Interval disabled — all channel widths

**Files:** `mac80211.c`, `mt7615/init.c`, `mt7915/init.c`

Short Guard Interval (SGI) reduces the inter-symbol guard period from 800 ns to
400 ns, producing approximately 11% higher peak throughput. In multipath
environments — where the signal reaches the receiver via several reflected paths
at slightly different delays — 400 ns is sometimes not enough time for the
energy from the previous symbol to decay before the next one arrives. This
causes inter-symbol interference (ISI) and frame decoding failures.

The fix removes `IEEE80211_HT_CAP_SGI_20`, `IEEE80211_HT_CAP_SGI_40`,
`IEEE80211_VHT_CAP_SHORT_GI_80`, and `IEEE80211_VHT_CAP_SHORT_GI_160` from the
capability advertisement in `mt76_init_sband()` and from the chip-specific
overrides in the MT7615 and MT7915/MT7916 init paths.

Removing these flags from the advertised capabilities causes the AP to negotiate
Long GI at association time. The AP will not use SGI because the adapter has
told it we do not support it. This applies to all channel widths (20, 40, 80,
and 160 MHz).

**Trade-off:** ~11% lower peak PHY throughput in return for reliable frame
decoding in multipath conditions.

---

### 2. Scan channel dwell time raised — legacy connac chips

**File:** `mt76_connac_mcu.c` — `MT76_CONNAC_SCAN_CHANNEL_TIME`

**Change:** 60 ms → 100 ms

When scanning for an AP — both during initial association and during reconnection
— the driver spends this long on each channel waiting for probe responses. At
60 ms, a loaded or slow AP may not respond in time before the scan moves to the
next channel. At 100 ms the driver is more likely to receive responses from APs
that are momentarily busy or running slower firmware.

Passive scans (no SSID specified) already double this value automatically, so
they move from 120 ms to 200 ms per channel.

**Applies to:** MT7603, MT7615, MT76x0, MT76x2. Connac2+ chips (MT7921,
MT7925, MT7996) pass 0 to the firmware and let it decide — those are unaffected.

**Trade-off:** Scans take ~67% longer per channel. A full 13-channel 2.4 GHz
scan takes ~1.3 s instead of ~0.8 s. This is only relevant during scanning, not
during normal operation.

---

### 3. Hardware retry limits raised to maximum — MT7603

**File:** `mt7603/init.c`

| Field | Before | After | Hardware max |
|---|---|---|---|
| `MT_AGG_RETRY_CONTROL_RTS_LIMIT` | 15 | 31 | 31 (5-bit field) |
| `MT_AGG_RETRY_CONTROL_BAR_LIMIT` | 1 | 15 | 15 (4-bit field) |

RTS_LIMIT controls how many times the hardware retries an RTS (Request to Send)
frame before giving up on the protected data frame. BAR_LIMIT controls retries
for Block ACK Request frames, which are used to set up and recover A-MPDU
aggregation sessions.

Raising both to their hardware maxima means the chip exhausts every available
attempt before reporting a failure upward. In congested RF environments where
management frame collision rates are high, this dramatically reduces the number
of failed transmissions that result in actual link failures.

---

### 4. RTS retry limit raised — MT76x2

**File:** `mt76x2/init.c` — `MT_TX_RTS_CFG`

**Change:** Retry byte `0x20` (32) → `0x7f` (127)

The `MT_TX_RTS_CFG_RETRY_LIMIT` field (bits 7:0 of `MT_TX_RTS_CFG`) controls
how many times the hardware retransmits an RTS frame before abandoning the
associated data frame. The field is 8 bits wide (maximum 255). Raising from 32
to 127 gives the hardware substantially more attempts before reporting failure in
congested deployments.

---

### 5. Rate adaptation patience increased — MT7603, MT7615

**Files:** `mt7603/mt7603.h`, `mt7615/mt7615.h`

| Constant | Before | After |
|---|---|---|
| `MT7603_RATE_RETRY` | 2 | 4 |
| `MT7615_RATE_RETRY` | 2 | 4 |

This constant controls how many consecutive transmission attempts the rate
adaptation algorithm allows at a given MCS (Modulation and Coding Scheme) before
stepping down to a lower, more robust rate. At 2, a single brief interference
burst causing two consecutive failures immediately triggers a rate downgrade.
Subsequent downgrade steps can cascade until the adapter is transmitting at a
fraction of its capable rate.

At 4, a transient interference event must cause four consecutive failures at the
same rate before a downgrade is triggered. This substantially reduces spurious
rate degradation during momentary congestion without meaningfully slowing the
rate adaptation response to genuine signal deterioration.

---

### 6. Watchdog patience extended — MT7603

**File:** `mt7603/mt7603.h` — `MT7603_WATCHDOG_TIMEOUT`

**Change:** 10 checks → 15 checks (100 ms each, so 1.0 s → 1.5 s)

The MT7603 hardware watchdog counts consecutive 100 ms intervals where the MAC
appears stalled (TX queue not draining, no RX activity) before triggering a
hardware reset. A reset causes a complete brief disconnection and re-association.

In congested RF environments, a real but heavily congested link can appear
stalled to the watchdog for over a second while frames are being retried. Raising
the threshold from 10 to 15 checks gives the hardware 50% more tolerance for
congestion-induced stalls, reducing resets that are triggered by heavy load
rather than actual hardware failure.

---

### 7. Power-save entry delay extended — MT7615, MT7921, MT7925

**Files:** `mt7615/mt7615.h`, `mt792x.h`

| Constant | Before | After |
|---|---|---|
| `MT7615_PM_TIMEOUT` | `HZ/12` (~83 ms) | `HZ/4` (250 ms) |
| `MT792x_PM_TIMEOUT` | `HZ/12` (~83 ms) | `HZ/4` (250 ms) |

This is the idle time after the last packet before the driver schedules the
firmware power-save entry work. At 83 ms, even a brief pause in traffic — a
single HTTP request waiting for a server response — causes the chip to enter
low-power mode. Entering power-save mode and waking from it both take time and
can cause the chip to miss beacons during the transition.

At 250 ms, the chip stays fully awake through typical interactive traffic gaps.
Power draw increases slightly during idle periods; connection stability improves
because the chip is more reliably present to receive beacons and management
frames.

This value can be overridden at runtime without recompiling via debugfs:

```bash
# View current value (in milliseconds)
cat /sys/kernel/debug/ieee80211/phy0/mt76/pm_idle_timeout

# Set to 500 ms
echo 500 > /sys/kernel/debug/ieee80211/phy0/mt76/pm_idle_timeout

# Disable power save entirely (set to a very large value)
echo 100000 > /sys/kernel/debug/ieee80211/phy0/mt76/pm_idle_timeout
```

Replace `phy0` with your interface's phy if different (`iw phy` to list them).

---

## What Was Not Changed

- **A-MPDU aggregation** — kept enabled. Disabling it collapses throughput to
  approximate 802.11g speeds with no meaningful connectivity benefit.

- **Beamformee support** — kept enabled. This adapter operates as a
  *beamformee*, meaning the AP steers its transmission toward us using our
  feedback. Disabling this would cause the AP to use omnidirectional
  transmission, reducing received signal strength directly.

- **Connac2+ scan dwell** — the MT7921, MT7925, and MT7996 pass 0 to firmware,
  which manages dwell internally. Overriding that value risks conflicting with
  firmware-internal scan policy.

- **mac80211-layer parameters** — auth/assoc retry counts, beacon loss
  tolerance, and association timeouts are owned by the kernel's mac80211 stack
  (`net/mac80211/mlme.c`). They cannot be changed in the driver without patching
  the kernel itself.

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

This produces `.ko` files in each chip subdirectory alongside the core modules
in the root directory. A successful build ends with output similar to:

```
  LD [M]  /path/to/mt76/mt76.ko
  LD [M]  /path/to/mt76/mt7603/mt7603e.ko
  ...
```

If you see errors about missing headers, verify that `linux-headers-$(uname -r)`
is installed and that `/lib/modules/$(uname -r)/build` exists and is not empty.

---

## Loading the Custom Driver

### Identify your chip

First, identify which mt76 module your adapter uses:

```bash
# For PCIe adapters
lspci -k | grep -A3 "Network\|Wireless"

# For USB adapters
lsusb
# Then check which module is currently bound:
lsmod | grep mt76
```

The module name tells you which `.ko` file to load. Common mappings:

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
# Replace mt7921e with your chip's module name
sudo modprobe -r mt7921e
# Core library modules unload automatically once nothing depends on them
```

If the interface is in use (connected to a network), disconnect first or bring
the interface down:
```bash
sudo ip link set wlan0 down
sudo modprobe -r mt7921e
```

### Load the custom modules

The core `mt76.ko` must be loaded before any chip module. If the connac library
is needed (MT7615, MT7921, MT7925, MT7996), load it after the core.

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

For USB chips, also load the USB bridge module:
```bash
sudo insmod mt76-usb.ko
# For MT792x USB:
sudo insmod mt792x-usb.ko
sudo insmod mt7921/mt7921u.ko
```

The interface should reappear within a few seconds. Confirm with `ip link` or
`iw dev`.

### Making the change persistent across reboots

Create a directory for the custom modules and install them:

```bash
# Create a version-specific override directory
KVER=$(uname -r)
sudo mkdir -p /lib/modules/$KVER/updates/mt76

# Copy all built modules into it
find . -name "*.ko" | sudo xargs -I{} cp {} /lib/modules/$KVER/updates/mt76/

# Rebuild the module dependency database
sudo depmod -a

# The kernel will now prefer modules in updates/ over the built-in ones
# Reboot to verify, or reload the modules manually as above
```

The `updates/` directory is searched before the kernel's own module tree, so
the custom modules take precedence automatically after `depmod -a`.

To confirm the right module is loaded after a reboot:
```bash
modinfo mt76 | grep filename
# Should point to /lib/modules/.../updates/mt76/mt76.ko
```

### Reverting to the mainline driver

Remove the installed files and rebuild the module index:

```bash
KVER=$(uname -r)
sudo rm -rf /lib/modules/$KVER/updates/mt76
sudo depmod -a
sudo modprobe -r mt7921e   # your chip module
sudo modprobe mt7921e      # reloads the mainline version
```

---

## Verifying the Changes Are Active

To confirm SGI is disabled after loading the module, check the station info
when connected to an AP:

```bash
iw dev wlan0 station dump
# Look for: tx bitrate ... HT/VHT
# You should NOT see "short GI" in the bitrate line
```

To confirm the scan dwell change, capture a scan with timing:
```bash
time iw dev wlan0 scan | grep -c "^BSS"
# Scans will take slightly longer than before
```
