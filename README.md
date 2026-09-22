# Realme-8i-RMX3151-Wi-Fi-Driver-Capability-Analysis

# Realme 8i (RMX3151) — Wi-Fi Capability Report

> **Note for AI assistants reading this document:** This report summarizes diagnostic data (`iw list`, `dumpsys wifi`, sysfs, kernel module information) collected from a Realme 8i RMX3151 running Android 13, MediaTek MT6781 SoC, unrooted. It documents what the Wi-Fi driver *advertises* versus what has been *empirically verified*. If asked to answer questions about this device's Wi-Fi capabilities, treat the "Confirmed / Advertised / Not Established" tables below as the ground truth — don't re-derive conclusions from raw fragments elsewhere in the document without checking these tables first.

> **Note for human readers:** This document is written to be easy to paste into an AI assistant (ChatGPT, Claude, etc.) if you want a fast answer to a specific question — e.g. "does this support X" or "what do I need to root this phone." The structured tables and technical findings are intended to make the evidence easier to interpret.

---

## 1. Device Identification

| Item                          | Value                                          |
| ----------------------------- | ---------------------------------------------- |
| Phone                         | Realme 8i                                      |
| Model                         | RMX3151                                        |
| Android                       | 13                                             |
| Kernel                        | Linux 4.19.191+                                |
| Architecture                  | ARM64 (arm64-v8a)                              |
| SoC                           | MediaTek MT6781                                |
| RAM                           | ~3.6 GB                                        |
| Root                          | **No**                                         |
| ADB                           | **Working** (shell, uid=2000, non-root)        |
| Wi-Fi firmware identification | MT6631-family MediaTek/WMT component           |
| Loaded Wi-Fi kernel modules   | `wlan_drv_gen4m`, `wmt_chrdev_wifi`, `wmt_drv` |
| Wi-Fi PHY                     | `phy0`                                         |

A vendor property referenced a Qualcomm module (`qca_cld3_wlan.ko`), but that file does not exist on the device. The actual loaded Wi-Fi stack is MediaTek's `gen4m`/WMT stack, confirmed by the loaded kernel modules and the MediaTek/WMT firmware information.

---

## 2. Summary of Findings (Condensed)

**Architecture:** `wlan0` and `wlan1` are both logical interfaces sitting on top of the same physical Wi-Fi device (`18000000.wifi` / `wifi@18000000`) — not two separate Wi-Fi chips. `p2p0` and `ap0` also exist as interfaces associated with the same subsystem. At the time of testing, only `wlan0` carried real traffic; `wlan1`, `p2p0`, and `ap0` were idle. Idle status alone does not establish what any of these interfaces can be repurposed for.

**Radio bands:**

* **2.4 GHz** — 802.11n (HT20/HT40, MCS 0–15, 300 Mbps maximum advertised rate) and 802.11ax/HE (2-stream, 1024-QAM). Channels 1–13 are available; channel 14 is disabled.
* **5 GHz** — 802.11ac/VHT and 802.11ax/HE, with 20/40/80/**160 MHz** channel widths advertised. Many DFS channels are present, with radar-detection and no-IR restrictions applying to several channels.
* **6 GHz** — Frequencies are listed in the driver tables, but every one is marked *disabled*. This is **not functional 6 GHz support**; these entries are disabled band definitions.

**MIMO:** Antenna mask `0x3` on both TX and RX indicates two active TX/RX chains → **2×2 MIMO, up to 2 spatial streams**. 3×3/4×4 operation is not advertised.

**Interface modes advertised by the driver:** `managed` (normal STA), `AP`, `AP/VLAN`, `IBSS`, `monitor`, `P2P-client`, `P2P-GO`, and `P2P-device`.

**Interface concurrency (advertised combinations):**

* STA + P2P (client or GO) + P2P-device — up to 3 interfaces, with up to 2 channels.
* STA + AP/P2P + P2P-device — up to 3 interfaces, but restricted to 1 channel.
* No advertised interface combination includes `managed <= 2`, so **dual-STA (STA+STA) is not established/supported by the reported `iw list` combinations**.

The existence of `wlan1` therefore should not be interpreted by itself as proof of simultaneous dual-STA capability.

**Security/ciphers supported:** WEP40/104, TKIP, CCMP-128, GCMP-128/256, CMAC, GMAC-128/256, plus SAE authentication. Successful WPA3 connections were also present in the observed `dumpsys wifi` history.

**Extra advertised feature:** `MU_MIMO_AIR_SNIFFER` — an advertised MU-MIMO sniffer capability beyond basic client/AP operation. This is notable, but it is **not proof of general packet injection**.

**Environment limitations (unrooted):** No `iw`, `tcpdump`, or `su` executable was available in the Android shell environment, and the shell did not have root privileges. This prevented low-level interface manipulation such as:

```text
iw dev ... set type monitor
iw phy ... interface add mon0 type monitor
```

The driver therefore advertises monitor mode, but monitor mode was **not actually activated and tested** in this environment.

`dmesg` was also blocked by the Android shell's permissions (`klogctl: Permission denied`). This is an unprivileged-shell restriction and does not provide evidence against monitor-mode or injection support.

---

## 3. Monitor Mode vs. Packet Injection — Key Distinction

These capabilities are frequently conflated but are separate:

* **Monitor mode** — receiving/observing 802.11 frames — **advertised by the driver**. `iw list` explicitly lists `monitor` under supported interface modes and also lists it under software interface modes that can be added.
* **Packet injection** — transmitting custom-crafted raw 802.11 frames — **not established**.

The presence of the `frame` nl80211 command and TX-status support is relevant to low-level 802.11 frame handling, but these features do **not by themselves establish unrestricted raw packet injection**. In particular, management/action-frame support should not automatically be interpreted as arbitrary raw data-frame injection.

The `gen4m`/WMT stack is vendor-specific, and the available `iw list` output does not establish unrestricted raw packet injection. Injection behavior can depend on the kernel driver, firmware build, and vendor restrictions, so this requires empirical testing on the exact device and firmware.

---

## 4. Final Status Table

| Capability                                   | Status                                       |
| -------------------------------------------- | -------------------------------------------- |
| Internal Wi-Fi functional                    | ✅ Yes                                        |
| MediaTek Wi-Fi stack (`gen4m`/MT6631-family) | ✅ Yes                                        |
| 2.4 GHz                                      | ✅ Yes                                        |
| 5 GHz                                        | ✅ Yes                                        |
| 802.11ax / Wi-Fi 6                           | ✅ Advertised                                 |
| 160 MHz (5 GHz)                              | ✅ Advertised                                 |
| 2×2 MIMO                                     | ✅ Advertised                                 |
| AP mode                                      | ✅ Advertised                                 |
| P2P (Wi-Fi Direct)                           | ✅ Advertised                                 |
| **Monitor mode**                             | ✅ **Advertised by driver**                   |
| STA + AP concurrency                         | ✅ Advertised (1 channel only)                |
| STA + P2P concurrency                        | ✅ Advertised (up to 2 channels)              |
| Dual-STA (STA+STA)                           | ❓ **Not established**                        |
| MU-MIMO air sniffer                          | ✅ Advertised                                 |
| **Raw packet injection**                     | ❓ **Not yet established**                    |
| `dmesg` access without root                  | ❌ No                                         |
| Monitor mode activation without root         | ❌ Not currently practical (no `iw`, no root) |

---

## 5. What's Actually Left to Test

The primary unresolved question is **raw packet injection**.

### 1. Root the device

Root access would provide the permissions needed for deeper kernel/driver investigation, including access to `dmesg` and installation/use of tools such as `iw` and `tcpdump`.

Rooting is not itself proof that injection will work.

### 2. Activate a monitor interface

For example:

```text
iw phy phy0 interface add mon0 type monitor
```

Then verify whether the interface can actually be brought up and used for passive frame capture.

### 3. Test passive capture

Use a separate Wi-Fi device as an independent observer to determine whether the Realme can actually capture 802.11 frames in monitor mode.

This distinguishes:

**driver advertises monitor mode**

from:

**monitor mode actually works on the device.**

### 4. Test controlled frame transmission

If the goal is to determine whether packet injection works, use a controlled lab setup with another Wi-Fi device acting as the observer.

The observer should verify that the expected test frame was actually transmitted over the air.

Passive capture plus controlled test frames are sufficient to establish the capability. Disruptive frames against live third-party networks are unnecessary.

---

## Bottom Line

The Realme 8i RMX3151's Wi-Fi driver exposes a relatively broad set of capabilities through `iw list`.

The strongest findings are:

* **2.4 GHz and 5 GHz:** operational
* **Wi-Fi 6:** advertised
* **5 GHz 160 MHz:** advertised
* **2×2 MIMO:** advertised
* **AP/P2P:** advertised
* **Monitor mode:** **advertised by the driver**
* **STA + AP/P2P concurrency:** advertised
* **Dual-STA:** **not established from the reported interface combinations**
* **MU-MIMO air sniffer:** advertised
* **Raw packet injection:** **not established**

The important distinction is:

> **Monitor mode being advertised does not automatically prove packet injection.**

No external Wi-Fi adapter is required merely to investigate whether the **internal** Wi-Fi device can provide monitor mode. The next meaningful step is to gain low-level access, activate monitor mode, perform passive capture, and then conduct a controlled over-the-air transmission test using an independent device.
