# PonWrt 6.18 for ZN504XG-D & HG5585F (Airoha AN7581)

[简体中文](README.md) | English

[![GitHub Actions Build Status](https://img.shields.io/github/actions/workflow/status/wuziwei/ponwrt-ZN504-HG5585F/build-zn504.yml?branch=master&style=flat-square&logo=github)](https://github.com/wuziwei/ponwrt-ZN504-HG5585F/actions)
[![Latest Release](https://img.shields.io/github/v/release/wuziwei/ponwrt-ZN504-HG5585F?style=flat-square&logo=github)](https://github.com/wuziwei/ponwrt-ZN504-HG5585F/releases)
[![Target](https://img.shields.io/badge/SoC-Airoha%20AN7581-blue?style=flat-square)](https://github.com/wuziwei/ponwrt-ZN504-HG5585F)
[![Device](https://img.shields.io/badge/Device-ZN504%20%26%20HG5585F-green?style=flat-square)](https://github.com/wuziwei/ponwrt-ZN504-HG5585F)
[![Kernel](https://img.shields.io/badge/Kernel-Linux%206.18-informational?style=flat-square)](https://github.com/wuziwei/ponwrt-ZN504-HG5585F)
[![License](https://img.shields.io/badge/License-GPL--2.0-orange?style=flat-square)](LICENSE)

Customized and performance-enhanced [PonWrt](https://github.com/pbs05/ponwrt) (based on [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) / Linux `6.18`) firmware engineered for dual flagship Airoha AN7581 optical gateways:
* **ZN504XG-D (`znxt_zn504xg-d`)**: Pure wired 2.5G high-performance ONT (EN8811H 2.5G port + 3×GbE)
* **FiberHome HG5585F CT (`fiberhome_hg5585f-ct`)**: Wi-Fi 6 ONT (Airoha AN7581 + MT7916D DBDC AX3000 + NPU WED hardware acceleration, native USB 2.0 configuration)

Every release automatically compiles and packages **both devices** into separate flashable images!

---

## 💡 I. Exclusive Optimization vs Upstream / Community Matrix

The table below details the technical generational gap between our enhanced PonWrt release and official upstream / generic community builds:

| Dimension / Feature | Upstream PonWrt / ImmortalWrt Stock | Generic Community Builds | **This Project (ZN504 & HG5585F Enhanced)** | Technical Advantage & Measured Benefits |
| :--- | :--- | :--- | :--- | :--- |
| **IPTV Error 1302 Root Fix** | ❌ Unaddressed. Bridge shares multicast & service ports with dynamic learning | ⚠️ Suggests periodic reboots or unicast hacks; adding `isolate` alone still flips FDB | **Exclusive Patch `03-pon-userspace-iptv-bridge-mac-fix.patch`**: Disables MAC learning on multicast ports (`learning='0'`) + dynamically discovers and statically pins BNG MAC (`bridge fdb replace ... static`) | **Permanently eliminates FDB table flipping and unicast black holes**; STB authenticates in seconds with zero 1302 timeouts over days of continuous playback |
| **HG5585F Wi-Fi 6 & NPU WED Offload** | ❌ Basic target only. Missing EEPROM extraction, misaligned MAC, missing WED firmware | ⚠️ Missing WED acceleration; SoftIRQ piles up on Core 0; speeds capped at 500–670 Mbps | **Integrated MT7916D + 8-core NPU WED 1200M driver**, calibration tables, `/usr/sbin/pin-wifi-cpus.sh` (`mt76-tx* -> CPU3`, 6 `napi` workers spread over `CPU0/1/2`, Ethernet IRQ 17/21 affinity) | **iPerf3 benchmarked at 1144 Mbps Downlink / 1200 Mbps Uplink** (peak 1314+ Mbps); perfectly balanced 4-core utilization with zero SoftIRQ accumulation |
| **Cross-Firmware PON Credentials Auto-Backup** | ❌ Upgrades wipe `/etc/config/pon`, bricking carrier link unless manually saved | ⚠️ Requires manual SSH backup and manual parameter restoration after flashes | **Bundled `/usr/sbin/backup-fiberhome-pon.sh` & `restore-fiberhome-pon.sh`**, automatically backing up credentials and injecting them into `/etc/sysupgrade.conf` | **Zero configuration loss on upgrade**; preserves LOID, PON SN, Vendor ID, and broadband PPPoE credentials seamlessly with one-click rollback |
| **NNdroid OLT Diagnostic & Link Hardening** | ❌ Lacks cold-start laser settling; premature `SYS_TX_READY` reports cause OLT kicks; fixed right-padding only | ⚠️ Strict OLTs (Huawei/ZTE) reporting `reason=0x01` require painful packet captures | **Integrated EN7572 8-consecutive-sample (140ms settling) laser latch protection**; Web UI toggle for Left vs Right LOID 0x00 padding; live Wire Bytes hex preview; Web PON MAC override | **100% stable cold-start link establishment**; unlocks challenging carrier OLTs with real-time wire-level frame inspection without packet sniffers |
| **QWRT 16MB Network Buffers & Queue Tuning** | ⚠️ Kernel default 2MB socket buffers, `netdev_max_backlog=1000`, `netdev_budget=300` | ⚠️ Missing comprehensive buffer tuning; burst drops on switch CDM queues during P2P | **Unified 16MB (`16777216`) TCP/UDP core & socket buffers**, `netdev_max_backlog=16384`, `netdev_budget=600`, `dev_weight=128`, `vm.percpu_pagelist_high_fraction=128`, `txqueuelen=5000` | **Completely eliminates multi-core memory allocation latency and CPU CDM1 burst drops**; guarantees butter-smooth 4K/8K multicast and multi-gigabit throughput |
| **LAN Broadcast Bypass (`NOTRACK`)** | ❌ UPnP SSDP and Windows WSD broadcasts tracked in conntrack table | ❌ No bypass; heavy LAN discovery traffic swells conntrack state tables and CDM queues | **Pre-configured SSDP (`239.255.255.250:1900`) and WSD (`3702`) `NOTRACK` rules** across IPv4 and IPv6 | **Insulates conntrack tables from broadcast storms**, reducing CPU lookup overhead and protecting hardware switch queues |
| **Plug-and-Play `br-iptv` + `rtp2httpd` Dual-Stack** | ❌ No pre-built bridge or IPTV proxy; requires manual VLAN, bridge, and proxy setup | ⚠️ Common legacy single-stack udpxy; prone to L2 multicast echo loops and port conflicts | **Pre-configured `br-iptv` asymmetric VLAN bridge (`isolate='1'` + `learning='0'`)**, `rtp2httpd` on bare port 4022 (dual-stack, 4 multi-core workers), `/www/iptv.m3u` & Web player | **Zero-touch plug-and-play**; browser, mobile, and STB playback without extra client software; completely prevents L2 loopback reflection |
| **Software Flow Offloading & Anti-IPv6 Regression Strategy** | ⚠️ Blindly enabling hardware flowtable offload (`flow_offloading_hw=1`) triggers an AN7581 PPE driver defect under IPv6 PPPoE, causing Quark/Alibaba CDN timeouts | ⚠️ Fully disabling flow offloading increases CPU utilization for bridge and routing | **Benchmarked Best Practice: enable software flow offload (`flow_offloading='1'`), disable PPE hardware offload (`flow_offloading_hw='0'`), disable `packet_steering`** | **Line-rate bridging & routing via kernel flowtable** without PPE IPv6 MTU drop regressions (restoring Quark App/Web 19.6MB/s downloads) while preserving 4-core Wi-Fi IRQ affinities |
| **First-Principles Pure Crypto Pipeline** | ⚠️ Retains SafeXcel / EIP-93 crypto coprocessor, causing severe DMA penalties | ⚠️ Retains hardware crypto driver, inducing Crypto API priority inversion and SoftIRQ lock contention | **Following ImmortalWrt official practice (`da63f7d`), completely removed `kmod-crypto-hw-eip93`**; dedicates 4x 1.2GHz Cortex-A53 NEON vector cores | **Zero DMA cache flushes and zero hard IRQ preemption**; WireGuard (ChaCha20-Poly1305) and Sing-Box pure Go-assembly NEON achieve lowest latency |
| **Sing-Box Native NFQUEUE Redirection** | ❌ Missing kernel `kmod-nft-queue`, causing `-EINVAL` crash when enabling `auto_redirect: true` | ⚠️ Relies on fragile custom nftables scripts prone to system rule conflicts | **Native Netfilter NFQUEUE kernel drivers, pre-bundled `sing-box 1.15.0-alpha.13` core** and Momo core-only mode (`core_only='1'`) supporting `endpoints[].on_demand` | **Native TUN zero-copy transparent redirection**; traffic routing and bypass adhere strictly to user subscriptions without rule friction |
| **Modern CI/CD Build Pipeline** | ⚠️ Basic single-runner workflow without compiler fingerprinting or multi-tier cache | ⚠️ Long build times (2–4 hours); fragile APT mirror dependencies cause frequent CI aborts | **Adopted 1018ji/build-ponwrt-daily innovations**: Ubuntu 24.04 + Node 24, `CCACHE_COMPILERCHECK` fingerprint locking, 5GB cache reuse, dynamic mirror probing, yq | **Incremental builds reduced to 20–35 minutes**; >99% build reliability; dynamic mirror probing ends GitHub Actions APT timeouts |

---

## 🚀 II. Exclusive Features & Deep Dives

### 1. 🌟 Root Fix: Carrier IPTV Error 1302 & Bridge MAC Table Flipping

#### ① Symptom & The Trap
When connecting carrier 4K/8K IPTV set-top boxes (ZTE, Huawei, Skyworth) through OpenWrt bridged VLANs:
* **Initial Playback Works**: The box boots up, EPG displays properly, and live TV streams smoothly.
* **Periodic Drops**: After 5–15 minutes, or immediately upon changing channels or accessing VOD, playback freezes with: **"Error Code 1302 / Authentication Failed / Server Response Timeout / Network Anomaly"**.
* **Temporary Reboot Illusion**: Rebooting the modem or reconnecting the Ethernet cable temporarily restores playback, but the error unfailingly recurs after another 5–15 minutes.

#### ② Deep Architectural Root Cause: Why Multicast Strangles Unicast
Carrier IPTV infrastructures operate across two distinct traffic planes:
* **Service Plane (Unicast VLAN, e.g. 301 / 41)**: Carries DHCP IP leasing, HTTP authentication, periodic heartbeat keep-alives, EPG navigation, time-shift, and VOD via service port `ct-iptv-svc` to the carrier BNG / authentication server.
* **Multicast Plane (Multicast VLAN, e.g. 302 / 45 / 50)**: Carries high-bitrate UDP broadcast streams and IGMP join/leave requests via multicast port `ct-iptv-mc`.

When both ports and the STB LAN port are bridged into a single bridge (`br-iptv`):

```
       [ Carrier BNG / Aggregation Switch ]
        (Shares a single physical MAC: BNG_MAC)
               /               \
       Service VLAN (301)     Multicast VLAN (302)
             /                   \
       (Unicast Channel)       (Downstream Stream/IGMP)
           /                       \
   +---------------------------------------+
   |        Linux Bridge (br-iptv)         |
   |                                       |
   |  ct-iptv-svc            ct-iptv-mc    |
   |  (Service Port)         (Multicast)   |
   +---------------------------------------+
                      |
                   lan2 / lan3
                      |
              [ Telecom 4K STB ]
```

1. **Shared BNG MAC Address**: To minimize hardware costs, carrier BNG routers frequently reuse the **exact same source MAC address (`BNG_MAC`)** for both the unicast service gateway and the multicast stream feed!
2. **Bridge Dynamic MAC Learning**: Standard Linux bridge ports default to `learning=1`. When an incoming frame arrives on any port, the bridge reads its source MAC and maps `Source_MAC -> Current_Port` in the kernel FDB (Forwarding Database).
3. **FDB Table Flipping & Unicast Black Hole**:
   - At boot, the STB authenticates over `ct-iptv-svc`. The bridge records: `BNG_MAC -> ct-iptv-svc`. Authentication succeeds, channel streams.
   - Multicast video frames immediately flood into `ct-iptv-mc`, carrying source MAC `BNG_MAC`.
   - The Linux bridge instantly **overwrites and flips** the FDB entry: `BNG_MAC -> ct-iptv-mc`!
   - Minutes later, the STB transmits a periodic unicast authentication heartbeat or lease renewal targeted to `BNG_MAC`.
   - The corrupted bridge FDB directs this unicast frame out of the multicast port `ct-iptv-mc`!
   - **Silent Drop Black Hole**: Carriers enforce strict ingress security on multicast VLANs (allowing only downstream multicast or upstream IGMP). Unicast frames entering the multicast port are **silently dropped without ICMP responses**.
   - The STB times out waiting for acknowledgments and displays **Error 1302**!

#### ③ Why Generic Workarounds Fail
* **Rebooting**: Merely clears the transient FDB table; the next multicast packet flips it again in milliseconds.
* **Port Isolation (`isolate='1'`) Alone**: `isolate` only prevents horizontal L2 forwarding between member ports; it does NOT stop incoming frames from polluting the global bridge FDB table.
* **IP-level static routing**: Ethernet bridging forwards frames based strictly on L2 destination MAC addresses before IP routing occurs.

#### ④ Our Engineering Dual-Lock Solution (`03-pon-userspace-iptv-bridge-mac-fix.patch`)

```
[STB sends unicast heartbeat] ---> [br-iptv]
                                       |
                        Lookup: BNG_MAC pinned to ct-iptv-svc (master static)
                                       |
                  [100% egresses via ct-iptv-svc] ===> Auth Succeeded! (Zero Drops)

[Multicast frame on ct-iptv-mc] ---> [learning='0' disabled] ===> FDB Never Contaminated!
```

1. **Lock 1: Hard Disabled MAC Learning on Multicast Ports (`learning='0'`)**:
   In `br-iptv` bridge and port configurations, MAC learning is permanently disabled on multicast ports:
   ```uci
   set network.luci_iptv_mc_port=bridge-port
   set network.luci_iptv_mc_port.device='ct-iptv-mc'
   set network.luci_iptv_mc_port.learning='0'
   set network.luci_iptv_mc_port.isolate='1'
   ```
   Reinforced at runtime via kernel sysfs and `bridge link`:
   ```bash
   bridge link set dev ct-iptv-mc learning off
   echo 0 > /sys/class/net/br-iptv/brif/ct-iptv-mc/learning
   ```
2. **Lock 2: Dynamic BNG MAC Discovery & Static FDB Pinning**:
   On network boot, the runtime script probes the legitimate BNG gateway MAC from `ct-iptv-svc` and permanently anchors it in the bridge FDB:
   ```bash
   bng_mac="$(bridge fdb show dev ct-iptv-svc 2>/dev/null | awk '/master br-iptv/ && !/permanent/ && !/self/ {print $1; exit}')"
   [ -z "$bng_mac" ] && bng_mac="$(ip neigh show dev ct-iptv-svc 2>/dev/null | awk '/lladdr/ {print $5; exit}')"
   if [ -n "$bng_mac" ]; then
       bridge fdb replace "$bng_mac" dev ct-iptv-svc master static
   fi
   ```
3. **Verification Commands**:
   ```bash
   # 1. Confirm learning is disabled on multicast port (output must be 0)
   cat /sys/class/net/br-iptv/brif/ct-iptv-mc/learning

   # 2. Confirm BNG MAC is anchored permanently to ct-iptv-svc
   bridge fdb show dev ct-iptv-svc | grep -E "static|permanent"
   ```

---

### 2. ⚡ FiberHome HG5585F Wi-Fi 6 + NPU WED 1200M Hardware Acceleration

* **MT7916D + 8-Core NPU WED Offload**: Pre-bundled with `en7581_MT7916_npu_rv32.bin` and `en7581_MT7916_npu_data.bin`. The Airoha 8-core RISC-V NPU handles wireless RX DMA and WED TX pathways directly for wire-speed routing with zero CPU penalty.
* **Automated Factory EEPROM Extraction**: Integrated `/etc/hotplug.d/firmware/10-mt7916-an-eeprom` and `/etc/init.d/mt7916-an-eeprom` (executed at `START=08` prior to driver loading), extracting the 4KB RF calibration block from UBI `factory` offset `0x1000` with `0x7916` magic byte validation.
* **Independent Preinit MAC Calculation**: Preinit script `/lib/preinit/07_an7581_bsp_wifi_mac` reads base MAC and sets 2.4G (`base + 6`) and 5G (`base + 7`) hardware addresses.
* **RF Power & Beamforming Tables**: Bundled with `SingleSKU` and `mt7916-sku.dat` / `mt7916-sku-bf.dat`.
* **4-Core Thread & Ethernet IRQ Affinity (`/usr/sbin/pin-wifi-cpus.sh` + `/etc/hotplug.d/net/99-wifi-affinity`)**:
  - `mt76-tx*` transmit threads pinned exclusively to `CPU3`;
  - 6 `napi/phy0-0` receive workers balanced (`0,0,0,1,2,2`) across `CPU0 / CPU1 / CPU2`;
  - Ethernet hard IRQs pinned: `IRQ 17 -> CPU1`, `IRQ 21 -> CPU2`;
  - Benchmarked on HG5585F 5G Wi-Fi with iPerf3 (5 threads) at **1144 Mbps Downlink / 1200 Mbps Uplink** (peak **1314+ Mbps**).
* **Async Wi-Fi Scan Protection**: Patched `rpcd` (`133-rpcd-async-scan.patch`) isolates channel surveys to asynchronous child processes, preventing LuCI 504 timeouts.

---

### 3. 🛡️ PON Auth Credentials & Momo Configuration Auto-Backup

* **One-Click Backup & Restore**: Bundles `/usr/sbin/backup-fiberhome-pon.sh` and `/usr/sbin/restore-fiberhome-pon.sh` to extract, validate, and restore `FHTT`, `HG5585FV1.0`, LOID, and PON SN.
* **Automated Sysupgrade Protection**: Automatically injects `/etc/config/pon`, `/etc/fiberhome_pon_backup.uci`, and Momo service files into `/etc/sysupgrade.conf`. Upgrading via Web or CLI preserves all carrier configurations.

---

### 4. 🔍 Enhanced OLT (Huawei/ZTE) Compatibility

* **EN7572 Cold-Start Laser Latch Stabilization**: 8-consecutive-sample verification (140ms settling window) prevents premature `SYS_TX_READY` reports before laser stabilization, eliminating initial registration disconnects.
* **Configurable LOID Padding Alignment**: Web UI selector for Left vs Right 0x00 padding in the 24-byte LOID field resolves `reason=0x01` authentication rejects on specific Huawei OLTs.
* **Live Wire Bytes Inspection**: Real-time hex and printable ASCII display of raw frame attributes (`CTC ONU SN` and active `LOID`) directly in LuCI Status.
* **Web PON MAC Override**: Instant PON interface MAC cloning from the web management interface.

---

### 5. 🚀 QWRT 16MB Network Buffers, 16384 netdev Queue Depth & Low Jitter

* **16MB Unified TCP/UDP Socket Buffers**:
  Aligned `net.core.rmem_max=16777216`, `net.core.wmem_max=16777216`, `net.ipv4.tcp_rmem`, and `net.ipv4.tcp_wmem`, preventing bufferbloat and packet drops across multi-gigabit pipes.
* **High-Throughput Network Backlog**:
  `netdev_max_backlog=16384`, `netdev_budget=600`, `dev_weight=128`, draining QDMA ring buffers at 600 packets per SoftIRQ loop to eliminate CPU CDM1 burst drops.
* **Anti-Fragmentation Memory Scheduling**:
  Backported QWRT's `vm.percpu_pagelist_high_fraction = 128`, eliminating per-CPU page allocation stalls under multi-core concurrency.
* **SSDP / WSD NOTRACK Bypass**:
  Exempts UPnP SSDP (`239.255.255.250:1900`) and Windows WSD (`3702`) from conntrack, insulating state tables from broadcast storms.
* **Software Flow Offloading & Anti-IPv6 Regression Strategy**:
  Defaults `flow_offloading='1'` while retaining `flow_offloading_hw='0'`. Benchmarks confirm kernel software flowtable offloading delivers wire-speed forwarding with near-zero CPU load, while bypassing AN7581 PPE hardware driver MTU defects on IPv6 PPPoE that cripple Quark App and Alibaba CDN; keeps `packet_steering='0'` to preserve 4-core Wi-Fi IRQ affinities.
* **First-Principles Pure Crypto Pipeline**:
  Completely removed `kmod-crypto-hw-eip93` (per ImmortalWrt `da63f7d`), freeing all 4 Cortex-A53 cores (1.2GHz) for native NEON vector execution (WireGuard ChaCha20-Poly1305 and Sing-Box Go-assembly).

---

### 6. 📺 Plug-and-Play `br-iptv` Asymmetric VLAN Bridge + `rtp2httpd` (4022)

* **Asymmetric VLAN Bridge (`br-iptv`)**: Pre-bridges `pon0.45` (IGMP upstream) and `pon0.50` (multicast downstream) with `isolate='1'` and `learning='0'`, preventing L2 reflection loops.
* **High-Performance C Multicast Proxy (`rtp2httpd`)**:
  - Listens on bare port `'4022'` dual-stack (IPv4 & IPv6);
  - 4 `SO_REUSEPORT` multi-core workers, 16384 zero-copy buffers, and 2MB UDP `rcvbuf`;
  - Built-in web monitoring panel (`:4022/status`) and modern web player (`:4022/player`).
* **Auto-Synchronized Playlist**: Pre-bundled `/www/iptv.m3u` automatically synced to gateway LAN IP via `95-iptv-m3u-ip`.

---

### 7. ⚙️ Modernized CI/CD Build Pipeline (Inspired by 1018ji/build-ponwrt-daily)

* **Ccache Compiler Fingerprint Persistence**:
  Enforces `CCACHE_COMPILERCHECK: '%compiler% -dumpmachine; %compiler% -dumpversion'` to lock the target architecture and GCC/Clang version string, maintaining high cache hit rates across dynamic GitHub Actions runners.
* **5GB Incremental Cache Reuse**:
  Configures `CCACHE_MAXSIZE: 5G` with `actions/cache/restore@v4` and `actions/cache/save@v4` keyed to config hashes, reducing compilation time from 2–3 hours to 20–35 minutes.
* **DL Source Dependency Caching**:
  Persistently caches `dl/` based on `feeds.conf.default` and config hashes, guarding against upstream download failures.
* **Ubuntu 24.04 Dynamic Mirror Probing**:
  Probes Kernel.org, US Archive, Rackspace, Princeton, Ubuntu Archive, and Azure mirrors dynamically at launch, automatically switching to the lowest-latency active mirror.
* **Integrated `yq` YAML/JSON Tooling**:
  Native Go-based `yq` CLI integration for streamlined parsing and transformation of IPTV configs and Sing-box subscriptions.

---

## 📦 Firmware Downloads (Releases)

Download production-tested images from [GitHub Releases](https://github.com/wuziwei/ponwrt-ZN504-HG5585F/releases) (4 clean images per build):

| Target Device | Firmware Filename | Usage |
| :--- | :--- | :--- |
| **ZN504XG-D** | `ponwrt-airoha-an7581-znxt_zn504xg-d-squashfs-sysupgrade.itb` | **Standard Sysupgrade (Recommended)**, preserves settings |
| **ZN504XG-D** | `ponwrt-airoha-an7581-znxt_zn504xg-d-initramfs-recovery.itb` | **U-Boot Web Recovery / Unbrick** |
| **HG5585F (CT Telecom)** | `ponwrt-airoha-an7581-fiberhome_hg5585f-ct-squashfs-sysupgrade.itb` | **Standard Sysupgrade (Recommended)**, native USB 2.0 & MT7916 Wi-Fi |
| **HG5585F (CT Telecom)** | `ponwrt-airoha-an7581-fiberhome_hg5585f-ct-initramfs-recovery.itb` | **U-Boot Web Recovery / Unbrick** |

---

## 🛠️ Flashing & Upgrade Guide

> [!IMPORTANT]
> **Credential Preservation Warning**:
> Operating on carrier networks typically requires cloned FiberHome (`FHTT`) parameters.
> Always upgrade using configuration retention or run `/usr/sbin/backup-fiberhome-pon.sh` before upgrading!

### Method 1: SSH CLI Flashing (Recommended)

```bash
# 1. Upload the image to /tmp
scp -O <firmware-name>-squashfs-sysupgrade.itb root@192.168.1.1:/tmp/sysupgrade.bin

# 2. Verify signature and image integrity
ssh root@192.168.1.1 "sysupgrade -T /tmp/sysupgrade.bin"

# 3. Flash while preserving configuration (do NOT use -n)
ssh root@192.168.1.1 "sysupgrade -v -c /tmp/sysupgrade.bin"
```

The device will flash and reboot in 1–2 minutes.

### Method 2: LuCI Web Upgrade

1. Navigate to `http://192.168.1.1` in your browser.
2. Go to **System -> Backup / Flash Firmware**.
3. Select your device's `squashfs-sysupgrade.itb` image.
4. Ensure **Keep settings** is checked, then upload and confirm.

### Method 3: U-Boot Web Recovery

If the device fails to boot:
1. Disconnect power, hold down the **Reset button**, and reconnect power.
2. Continue holding Reset for 8–10 seconds until LEDs flash, then release.
3. Connect your PC to a LAN port, configure static IP `192.168.1.2` / `255.255.255.0`.
4. Open `http://192.168.1.1` in your browser to access the recovery web UI.
5. Upload the corresponding `initramfs-recovery.itb` image to restore.

---

## 📖 ZN504XG-D PON Provisioning & OLT Registration Guide (Field Lessons)

Replacing stock carrier optical modems (e.g. Huawei HS8145X6, FiberHome HG5585F) with the **ZN504XG-D** (Airoha AN7581) to unlock 2.5G full-rate connectivity is a primary use case.

### 1. Essential Parameters to Extract from Stock Modem

Before disconnecting your original optical modem, record the following credentials:

| Parameter | Description | Example (Telecom) |
| :--- | :--- | :--- |
| **PON SN** | 16-character PON Serial Number (first 4 characters are vendor ASCII, last 8 are hex) | `48575443XXXXXXXX` (`HWTC` Huawei) |
| **Broadband LOID** | Logic ID for telecom authentication | Phone number or `SZ10000XXXXXX` (leave password blank if none) |
| **Vendor & Model** | Vendor ID code and equipment hardware version | Vendor: `HWTC` or `FHTT`, Model: `HS8145X6` or `HG5585FV1.0` |
| **Internet VLAN ID** | 802.1q VLAN Tag bound to Internet service | Often `41`, `43`, or `46` (multicast/IPTV typically `45`) |
| **PPPoE Credentials** | Broadband dial-up username and password | `0755XXXXXXXX@163.gd` |

---

### 2. ZN504 Configuration Mapping

In LuCI (`http://192.168.1.1`), navigate to **Network -> PON Settings**:

#### ① Physical Link (Line 0)
- **Device**: `pon0`
- **Mode**: **`xgpon`** (standard for 10G PON Telecom connections; use `gpon` for Gigabit GPON, or `epon` for 10G-EPON)
- **Serial Number**: 16-character hex SN from original modem (e.g. `48575443XXXXXXXX`)

#### ② OMCI Protocol Identification (Line 0 OMCI)
- **Vendor ID**: **`FHTT`** (⚠️ See Lesson 1 below — recommended even if original modem was Huawei)
- **Equipment ID**: **`HS8145X6 (PON ONT)`** (⚠️ See Lesson 2 below — parentheses are mandatory)
- **Hardware Version**: `HG5585FV1.0`
- **Software Version**: `RP0100`
- **LOID**: Your broadband LOID (e.g. `SZXXXXXXXXX`)
- **Operator ID**: `CTC` (China Telecom)

#### Underlying Configuration (`/etc/config/pon`):
```uci
config xpon 'line0'
	option device 'pon0'
	option serial_number '48575443XXXXXXXX'
	option mode 'xgpon'

config omci 'line0_omci'
	option line 'line0'
	option device 'omci0'
	option omcc_version '0x86'
	option vendor_id 'FHTT'
	option equipment_id 'HS8145X6 (PON ONT)'
	option hardware_version 'HG5585FV1.0'
	option software_version 'RP0100'
	option loid 'SZXXXXXXXXXXXXX'
	option operator_id 'CTC'
```

---

### 3. ⚠️ Critical Field Lessons & Common Pitfalls

#### 🛑 Pitfall 1: Never report `HWTC` when upstream OLT is Huawei!
* **Symptom**: The modem momentarily enters O5 state, but within **100 milliseconds** the upstream Huawei OLT issues a `Deactivate_ONU-ID` command (msg-id `0x05`), falling into an endless `O1 -> O2 -> O4 -> O5 -> O2` reconnect loop.
* **Root Cause**: When an ONU reports `HWTC` (Huawei vendor code), Huawei OLTs enforce proprietary Huawei-only handshakes (bombarding OMCI ME Class 350 firmware image checks and Class 373 hardware fingerprinting). Because open-source PonWrt lacks Huawei proprietary closed-source response handlers, the OLT classifies the device as an illegitimate clone and terminates it.
* **Solution**: Change `vendor_id` to **`FHTT` (FiberHome)**. Carrier OLTs are required to support third-party hardware. Reporting `FHTT` prompts the Huawei OLT to abandon proprietary checks and gracefully fall back to standard ITU-T G.988 / China Telecom CTC specifications, smoothly registering via LOID.

#### 🛑 Pitfall 2: Equipment ID must include `(PON ONT)` suffix
* **Symptom**: Link enters O5, but the OLT refuses to provision GEM Ports / data channels.
* **Solution**: The ITMS / OLT equipment database requires the full string: **`HS8145X6 (PON ONT)`** (including space and parentheses).

#### 🛑 Pitfall 3: PPPoE must bind to the VLAN interface (`pon0.41`)
* **Symptom**: O5 state is stable, but WAN PPPoE discovery fails with continuous PADI timeouts.
* **Solution**: In **Network -> Interfaces**, bind WAN to **`pon0.41`** (or create an 802.1q device for your local VLAN).

#### 🛑 Pitfall 4: Carrier Session Lockout (`check user-name in black list`)
* **Symptom**: PPPoE handshake succeeds, but authentication fails with:
  > `Remote message: 28103012: check user-name in black list. failed!`
* **Solution**: Disconnecting without logging out leaves an active session on the BRAS. Wait 3–5 minutes with the fiber disconnected to allow stale sessions to expire.

#### 🛑 Pitfall 5: Configuration Loss Across System Upgrades
* **Symptom**: Losing `/etc/config/pon` after a clean firmware flash leaves the device disconnected.
* **Solution**: Run `/usr/sbin/backup-fiberhome-pon.sh` once. It archives your credentials to `/etc/fiberhome_pon_backup.uci` and adds them to `/etc/sysupgrade.conf`.

---

### 4. Verification & O5 Diagnostic Indicators

Run the following command to verify registration status:

```bash
pondctl status --line line0
```

**✅ Expected Output for Healthy Registration**:
```json
{
  "channel_available": true,
  "loid_configured": true,
  "authentication_status": 1,
  "authentication_meaning": "accepted",
  "olt_vendor_id": "HWTC",
  "backend_state": "applied",
  "vlan_ids": [1, 41, 45]
}
```

---

## 📱 III. Client IPTV Subscription & Management (Out-of-the-Box)

In TV boxes (APTV / TiviMate / Kodi), computers (PotPlayer / VLC / IINA), or phones on the LAN (replace `192.168.1.1` if you modified your LAN IP):

* **Recommended Dynamic M3U Playlist (`rtp2httpd`)**:  
  `http://192.168.1.1:4022/playlist.m3u`
* **Static Built-in Playlist (`uhttpd`)**:  
  `http://192.168.1.1/iptv.m3u`
* **Web Monitoring Dashboard**:  
  `http://192.168.1.1:4022/status`
* **Zero-Install Modern Web Player**:  
  `http://192.168.1.1:4022/player`
* **Single Stream URL Format**:  
  `http://192.168.1.1:4022/rtp/239.77.0.174:5146`

---

## 🚀 IV. Sing-Box Native `auto-redirect` Configuration

With `nfnetlink_queue` compiled in, Sing-Box handles redirection natively. Per [Sing-Box documentation](https://deepwiki.com/SagerNet/sing-box), `auto_redirect` requires `auto_route: true`:

```json
{
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": [
        "172.19.0.1/30"
      ],
      "auto_route": true,
      "strict_route": false,
      "auto_redirect": true
    }
  ]
}
```

---

## 🏗️ V. Building From Source

```bash
# 1. Clone repository & update feeds
git clone https://github.com/wuziwei/ponwrt-ZN504.git
cd ponwrt-ZN504
./scripts/feeds update -a && ./scripts/feeds install -a
bash ./scripts/customize.sh

# 2. Apply AN7581 enhanced configuration
./scripts/kconfig.pl + configs/an7581.config configs/release.config > .config
make defconfig

# 3. Compile
make download -j$(nproc)
make -j$(nproc) || make -j1 || make -j1 V=s
```

Outputs are generated under `bin/targets/airoha/an7581/`.

---

## ⚠️ VI. Disclaimer

1. This project is provided for technical exchange and networking research purposes.
2. Upgrading firmware carries inherent risks. Always backup your carrier credentials with `backup-fiberhome-pon.sh` before proceeding.
3. Observe all local telecommunication laws and regulations.

---

## 🙏 VII. Acknowledgments & Credits

This project builds upon the pioneering efforts of the open-source community:

- **[pbs05/ponwrt](https://github.com/pbs05/ponwrt)**: Founder of PonWrt; created the foundational Airoha AN7581 / AN7583 open-source architecture and native driver implementations.
- **[1018ji/build-ponwrt-daily](https://github.com/1018ji/build-ponwrt-daily)**: Modern CI/CD workflow pioneer; provided the architectural inspiration for Ccache compiler fingerprint persistence (`CCACHE_COMPILERCHECK`), 5GB multi-tier cache reuse, and dynamic mirror probing.
- **[LeoooChen/PonWRT-TYFZA1](https://github.com/LeoooChen/PonWRT-TYFZA1)**: Pioneer in hardware traffic offloading and 16MB ring buffer tuning; contributed line-rate PPPoE & XG-PON offload practices and `rpcd` async Wi-Fi scanning.
- **[Kahen/PonWrt-CI-UBI](https://github.com/Kahen/PonWrt-CI-UBI)**: Contributed AN7581 SoC thermal telemetry RPC (`temperature-rpc.uc`) and MTD read-write protection (`kmod-mtd-rw`).
- **[ImmortalWrt](https://github.com/immortalwrt/immortalwrt) & QWRT Communities**: Insights on Linux network stack tuning, eliminating EIP-93 crypto coprocessor negative acceleration, PPE flowtable acceleration, and Wi-Fi WED hardware offloading.
- **[NNdroid](https://github.com/NNdroid)**: For EN7572 laser latch stabilization, LOID padding alignment, and CTC raw wire byte inspection tools.
- **[OpenWrt Project](https://github.com/openwrt/openwrt)**: Foundation for embedded networking OS.
- **[stackia/rtp2httpd](https://github.com/stackia/rtp2httpd)**: High-performance zero-copy 4K/8K IPTV multicast, FCC, and RTSP proxy with built-in web player and status dashboard.
- **[SagerNet/sing-box](https://github.com/SagerNet/sing-box)**: Next-generation proxy engine.
- **[nikkinikki-org/OpenWrt-momo](https://github.com/nikkinikki-org/OpenWrt-momo)**: Streamlined sing-box WebUI and transparent gateway integration.
- **[rchen14b/luci-app-airoha-npu](https://github.com/rchen14b/luci-app-airoha-npu)**: Airoha AN7581 8-core NPU & PPE telemetry dashboard.
- **[VizzleTF/luci-theme-footstrap](https://github.com/VizzleTF/luci-theme-footstrap)**: Modern responsive theme for PonWrt.
- **[oenhu](https://github.com/oenhu)**: For PR #116 preventing redundant DHCPv6 `wan6` interfaces on DHCP networks.
