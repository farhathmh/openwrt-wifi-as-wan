<div align="center">

# OpenWrt — WiFi as WAN

**Building homelab connectivity from scratch with no owned infrastructure,
a second-hand router, and OpenWrt.**

![Status](https://img.shields.io/badge/Status-In_Progress-E57000?style=for-the-badge)
![OpenWrt](https://img.shields.io/badge/OpenWrt-00B5E2?style=for-the-badge&logo=openwrt&logoColor=white)
![Networking](https://img.shields.io/badge/Networking-VLANs%20%7C%20DNS%20%7C%20DHCP-00B4D8?style=for-the-badge)

</div>

---

## Quick Controls

| Jump | Purpose |
|------|---------|
| [Scenario](#the-scenario) | Understand the real-world constraint |
| [Hardware](#the-hardware-decision) | See why AX23 v1 was selected |
| [Problem](#the-network-problem) | Review rejected options |
| [Solution](#the-solution) | Follow the implementation path |
| [Outcome](#what-this-enabled) | Validate what changed |
| [Lessons](#what-i-learned) | Capture repeatable insights |

---

## Build Dashboard

| Signal | Status |
|--------|--------|
| Project State | ![State](https://img.shields.io/badge/State-Operational-0A9396?style=flat-square) |
| Router | ![Router](https://img.shields.io/badge/AX23-v1-E57000?style=flat-square) |
| Firmware | ![Firmware](https://img.shields.io/badge/OpenWrt-Stable-00B5E2?style=flat-square&logo=openwrt&logoColor=white) |
| WAN Mode | ![WAN](https://img.shields.io/badge/WAN-WiFi_Client-1D3557?style=flat-square) |
| LAN Mode | ![LAN](https://img.shields.io/badge/LAN-Wired_Distribution-457B9D?style=flat-square) |

---

## The Scenario

I live and work in Dubai as a Field IT Support Engineer. After months of
running VMware on a laptop — reformatting constantly, dealing with crashes,
struggling every time I wanted to test a new OS — I decided to build a
proper homelab.

The constraint: I live in a shared apartment. No owned internet connection.
Internet arrives via shared WiFi only. No access to the upstream router.
No ISP line of my own.

The goal: take that shared WiFi and route it into a fully functional wired
LAN to serve a multi-device homelab environment.

---

## The Hardware Decision

Before the network problem even surfaced, the first challenge was choosing
the right hardware on a tight budget.

Rackmount servers were immediately ruled out — too loud, too large for an
apartment, and unnecessary for the goal. After a month of research I settled
on the **HP Z440 Workstation** — old hardware, cheap on the second-hand
market, but natively supporting the **Xeon E5-2698 v4** processor and
**DDR4 ECC RAM**. A PC tower with the internals of several rack servers at
a fraction of the cost.

The unit I found came with an E5-1620 V3. The E5-2698 v4 upgrade came
later via AliExpress — 20 cores, 40 threads, sourced cheaply.

> Facebook Marketplace was the primary sourcing platform for all hardware
> in this lab.

---

## The Network Problem

Once the Z440 was running Proxmox, the network challenge became clear
immediately.

A WiFi USB dongle connected to the shared apartment WiFi could provide
internet — but it cannot function as a proper network bridge. A WiFi
adapter cannot forward multiple IPs from a single MAC address to multiple
downstream devices.

### Plans considered and why they were ruled out

| Option | Problem |
|--------|---------|
| NAT via Proxmox directly | Worked but messy — complex config, hard to troubleshoot, management IPs difficult to reach |
| OpenWrt as a Proxmox VM | Viable but still dependent on the USB dongle as WAN — same MAC address limitation, adds VM overhead |
| Buy a managed switch | Extra cost, doesn't solve the WAN source problem |
| Run a separate router | Needed the right hardware and firmware to support WiFi as WAN |

<details>
<summary><strong>Decision notes</strong></summary>

- Primary filter was reliability over convenience.
- Solutions dependent on USB dongles were deprioritized due to interface limits.
- Router-level routing provided cleaner troubleshooting than host-level NAT.

</details>

---

## The Solution

### Hardware

| Device | Source | Cost |
|--------|--------|------|
| TP-Link AX23 v1 | Facebook Marketplace | Less than the cost of an evening snack |

> Important: Only the **v1** of the TP-Link AX23 supports OpenWrt.
> Other hardware versions are not compatible. Verify the version before
> purchasing.

### Why OpenWrt

OpenWrt is an open source Linux-based firmware for routers. It supports
WiFi as WAN natively — the router connects to an upstream WiFi network
as a client and routes that connection to all wired LAN ports. This is
exactly what was needed.

### Installation

1. Download the correct OpenWrt image for TP-Link AX23 v1
2. Access the stock TP-Link firmware admin panel
3. Navigate to firmware upgrade
4. Upload the OpenWrt image
5. Wait for flash to complete — router reboots into OpenWrt

> Full step-by-step configuration guide → [docs/openwrt-setup.md](docs/openwrt-setup.md)

### Network Configuration

| Interface | Role |
|-----------|------|
| wlan0 (5GHz) | WAN — connects to shared apartment WiFi as upstream source |
| wlan1 (2.4GHz) | Broadcast — personal WiFi network for wireless device access |
| eth0-3 (LAN ports) | Wired LAN — all homelab devices connected here |

<details>
<summary><strong>Topology preview</strong></summary>

```text
Shared Apartment WiFi (upstream)
            |
        wlan0 (5GHz)
        TP-Link AX23
      /      |      \
 wlan1   LAN ports   LuCI
 2.4GHz   eth0-3   management
```

</details>

---

## Unexpected Benefits

This is what was not planned but came as a bonus.

The TP-Link AX23 has two completely independent WiFi radios — 2.4GHz
and 5GHz — that can operate as two separate interfaces simultaneously
in OpenWrt.

This meant:
- **5GHz** could be dedicated entirely to WAN (upstream connection) for
  maximum throughput
- **2.4GHz** could broadcast a personal WiFi network simultaneously

This eliminated the need for a second USB WiFi dongle that was originally
planned to provide wireless access to personal devices. It also eliminated
the need to use the TP-Link AX23's USB port to install dongle drivers —
a plan that was shelved entirely.

The result was a cleaner, simpler setup than originally designed — with
better performance and fewer points of failure.

---

## What This Enabled

With this solution in place, the entire homelab became fully operational:

- All wired devices (HP Z440, NUC1, NUC2, Raspberry Pi 4B) connected
  via LAN ports with proper DHCP
- Personal wireless devices connected via the 2.4GHz broadcast network
- Full internet access across all devices from a single shared WiFi source
- Clean network topology with a proper router handling all routing,
  DNS, DHCP, and firewall

<details>
<summary><strong>Operational checks</strong></summary>

- Verify internet reachability from at least one wired host.
- Verify DHCP lease allocation to wired and wireless clients.
- Verify DNS resolution through router for both networks.
- Verify firewall defaults and outbound connectivity.

</details>

---

## What I Learned

- OpenWrt installation and LuCI web interface navigation
- WiFi client mode configuration (WiFi as WAN)
- Independent dual-radio management on a single device
- DHCP and DNS server configuration on embedded Linux
- Practical NAT and firewall rules
- Evaluating hardware compatibility before purchasing (version-specific
  OpenWrt support)

---

## Part of Erenyx Lab

This repo documents one problem solved inside a larger homelab environment.

> Full lab documentation → [farhathmh/erenyx-lab](https://github.com/farhathmh/erenyx-lab)

---

<div align="center">
<sub>Built by <a href="https://github.com/farhathmh">Farhath Manas</a> — Dubai</sub>
</div>