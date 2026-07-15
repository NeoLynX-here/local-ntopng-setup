# ntopng Installation & Evaluation Report

## Overview

ntopng is a web-based, real-time network traffic monitoring and cybersecurity tool. Unlike a simple bandwidth counter, it inspects traffic (via nDPI, its bundled deep packet inspection library) to identify hosts, flows, and application-level protocols, and layers behavioral alerting on top of that visibility. This report documents the installation and functional evaluation of ntopng on a personal Linux laptop.

## Environment

| Item | Value |
|---|---|
| Host | Lenovo LOQ 15IRX9 (`lynx-loq-15irx9`) |
| OS | Ubuntu 26.04 LTS ("resolute") |
| Kernel | Upgraded during install: `7.0.0-14-generic` → `7.0.0-27-generic` |
| Monitored interface | `wlp8s0` (Wi-Fi) |
| ntopng version | Community v.6.7.260714 |

## Installation

Per [ntop's official installation guide](https://www.ntop.org/support/documentation/software-installation/) for Ubuntu 26.04 LTS (nightly channel — matches the date-stamped package versions seen below, e.g. `6.7.260714`), the documented process is three steps:

1. **Add the ntop repository:**
   ```
   apt-get install software-properties-common wget
   add-apt-repository universe
   wget https://packages.ntop.org/apt/26.04/all/apt-ntop.deb
   apt install ./apt-ntop.deb
   ```
2. **Refresh package lists:**
   ```
   apt-get clean all
   apt-get update
   ```
3. **Install the ntop suite:**
   ```
   apt-get install pfring-dkms nprobe ntopng n2disk cento ntap
   ```

### Core ntop packages

| Package | Version | Role |
|---|---|---|
| `ntopng` | 6.7.260714-28620 | Web UI / traffic analysis engine |
| `ntopng-data` | 6.7.260714 | Static data/assets bundle for ntopng (GeoIP DBs, categories, etc.) |
| `nprobe` | 11.1.260714-8982 | NetFlow/IPFIX probe and collector |
| `cento` | 2.5.260714-1267 | High-speed (1–100 Gbit) flow probe / traffic balancer |
| `n2disk` | 3.9.260714-5628 | Lossless packet-to-disk recorder |
| `ntap` | 1.3.260714-152 | Encrypted virtual tap for remote traffic mirroring |
| `pfring` / `pfring-dkms` | 9.3.0-10698 / 9.3.0.10698 | Kernel-level packet capture acceleration (DKMS kernel module) |
| `ndpi` | 5.1.0-5939 | Deep packet inspection / application classification library |
| `ntop-license` | 1.0-734 | Shared license manager service used by the commercial ntop tools |

### Services enabled

The install created and started systemd units for: `ntop-service`, `ntop-license-manager`, `cento`, `n2disk`, `nprobe`, `redis-server`, `pf_ring`, and `ntopng`. Per the post-install notices: `cento`, `n2disk`, and `nprobe` carry perpetual licenses with 1 year of included maintenance (after which the software keeps running, but further updates would need `apt-mark hold` on that package to avoid breaking it) — **ntopng Community itself requires no license.**

## What Each Component Does

- **ntopng** — the actual monitoring dashboard; everything below either feeds it or extends it to higher speeds/scales.
- **nDPI** — classifies traffic by application (e.g. distinguishing Cloudflare WARP from generic HTTPS) rather than by port number alone.
- **PF_RING** — kernel packet capture acceleration used by the other tools.
- **nprobe / cento** — NetFlow/IPFIX probes for feeding ntopng from a router/switch or from very high-speed (10–100 Gbit) links; not exercised in this test, since ntopng captured directly off `wlp8s0`.
- **n2disk** — full packet-to-disk recording for forensics; not exercised.
- **ntap** — encrypted remote virtual tap for containers/VMs; not exercised.
- **redis-server** — internal state cache for ntopng.

Only **ntopng Community** is free and fully functional out of the box. `nprobe`, `cento`, `n2disk`, and `ntap` are commercial products that run in a demo/limited mode without a purchased license — confirmed by the "Upgrade to Pro/Enterprise version" banner persistently shown in the ntopng UI throughout testing.

## Testing & Verification

| # | Test | Result |
|---|---|---|
| 1 | Confirm monitored interface | `wlp8s0` (Wi-Fi) actively bound, live throughput graph updating — not the placeholder System interface. |
| 2 | Generate mixed traffic | Speedtest + YouTube browsing generated a mix of bulk transfer, video, and DNS/mDNS traffic. |
| 3 | Hosts page | 36 hosts detected on the LAN across 4 pages, including phones, named laptops, and IoT devices — hostnames and traffic breakdown populated automatically without manual Network Discovery configuration. |
| 4 | Flow / application classification (nDPI) | Confirmed working at the protocol level, not just port-guessing: flows were labeled `QUIC.CloudflareWarp`, `MDNS`, and `TuyaLP` (a Tuya smart-device protocol) rather than generic `UDP`. |
| 5 | Active Monitoring | Configured 3 targets: ICMP to `1.1.1.1` (40 ms threshold), ICMP to the laptop itself (10 ms threshold), and a `speedtest.net` Speedtest measurement (190 Mbps threshold). The `1.1.1.1` check measured 143.46 ms — over threshold — and ntopng correctly flagged it with an alert icon, confirming threshold-based alerting works. |
| 6 | Alerting / anomaly detection | Alerts fired organically during testing (Alerts Explorer, Interface view): two **Ghost Networks** alerts and one **Slow Periodic Activity** alert (see findings below). |
| 7 | Network Discovery | Manually enabled (off by default on a fresh install) and run via **Run Discovery**. Identified 19 devices on the LAN using ARP/SSDP/MDNS/SNMP, resolving each to a manufacturer via MAC OUI lookup (e.g. correctly identifying the router's vendor), a hostname where one was broadcast, and a device-type icon — including distinct icons for Apple devices and Spotify Connect-capable devices. |

## Findings / Documentation Gaps

- **Ghost Networks alerts.** ntopng's behavioral alerting flagged traffic from two subnets as "not belonging to the wlp8s0 networks" during testing.
This alert type fires whenever ntopng sees traffic to/from a subnet that isn't in its configured **Local Networks** list — one instance was the laptop's own current LAN (a real gap, since the docs imply local networks are auto-detected from the interface's IP at startup, but that wasn't complete here); the other was simply stale residue from a mobile-hotspot connection used the previous day, not a live issue.
Worth reviewing **Settings → Networks → Local Networks** after install or after switching networks, since auto-detection doesn't always catch up on its own.
 
- **Slow Periodic Activity alert.** A "5second" internal Lua script ran over 1:05, longer than its own budget.
This is a system-level performance alert rather than a network one — plausibly caused by running ntopng alongside heavy foreground use (multiple browser tabs, speedtest) on laptop-class hardware rather than the dedicated appliance hardware ntop's docs typically assume.
Worth watching if it recurs under idle conditions.
 
- **Cloudflare WARP dominated traffic volume.** A single QUIC flow to `162.159.198.2` (520 MB sent) accounted for the bulk of observed traffic — nDPI correctly identified it as `QUIC.CloudflareWarp` rather than lumping it in as generic encrypted traffic, which is the clearest demonstration of DPI-based classification adding value over a plain bandwidth counter.

## Capabilities Overview (Confirmed Working — ntopng Community)

- Real-time dashboard: top talkers, top hosts, top applications, traffic classification, all on a live-updating interface.
- Automatic host discovery with hostname/device-type labeling, no manual configuration required.
- Active Network Discovery (ARP/SSDP/MDNS/SNMP) that identifies devices by manufacturer (MAC OUI lookup), hostname, and device type — must be manually enabled first, as it's off by default on a fresh install.
- Layer-7 application classification via nDPI (protocol-specific, e.g. distinguishing VPN/tunnel traffic and IoT protocols from generic traffic).
- Live flow explorer with per-flow throughput, byte counts, and ASN attribution (e.g. Cloudflare AS13335).
- Active Monitoring with configurable ICMP/Speedtest checks and per-target alert thresholds.
- Behavioral alerting (Ghost Networks, Slow Periodic Activity) surfaced without any manual rule configuration.

## Conclusion

ntopng Community is installed and fully functional on `wlp8s0`, with host discovery, DPI-based flow classification, active monitoring, and alerting all verified against live traffic. The commercial add-ons (`nprobe`, `cento`, `n2disk`, `ntap`) installed alongside it were not exercised and are not needed for this single-host use case.

<img width="2469" height="1393" alt="Screenshot From 2026-07-15 11-31-19" src="https://github.com/user-attachments/assets/18a9cccf-6cf6-4464-a5fb-c5ef4d0df646" />
<img width="2469" height="1393" alt="image(1)" src="https://github.com/user-attachments/assets/def0601c-9161-47ea-bc23-e26f26af6ff7" />
<img width="2469" height="1393" alt="image" src="https://github.com/user-attachments/assets/0d751869-73c2-4496-bb00-659fed809a28" />
<img width="2469" height="1393" alt="Screenshot From 2026-07-15 11-33-54" src="https://github.com/user-attachments/assets/7206de4a-2e0a-48c1-97a2-964882a6b232" />
<img width="2469" height="1393" alt="image(2)" src="https://github.com/user-attachments/assets/8bfa6a83-8651-4e4d-aa4f-2becc651cc7e" />
<img width="2469" height="1393" alt="Screenshot From 2026-07-15 12-14-38" src="https://github.com/user-attachments/assets/9ca001e5-ebf9-49a3-b758-41b31ed827b7" />
