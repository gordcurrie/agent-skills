---
name: audit-network-security
description: >
  Security audit of a UniFi home or small-office network using the unifi-mcp
  tools. Use this skill when asked to audit, review, check, or assess the
  security posture of a UniFi network — including device inventory, firmware
  currency, WiFi configuration, firewall rules, network segmentation, DNS
  anomalies, VPN configuration, and device health. Produces a prioritised
  findings report.
license: MIT
---

# Skill: Audit Network Security (UniFi via unifi-mcp)

This skill performs a structured security audit of a UniFi network using the
tools exposed by [gordcurrie/unifi-mcp](https://github.com/gordcurrie/unifi-mcp).
Work through each section below in order, collect findings, then produce a
**Prioritised Findings Report** at the end.

---

## Before you start

Confirm the unifi-mcp server is configured and the following tools are available:
`list_clients`, `list_devices`, `list_firewall_policies`, `list_acl_rules`,
`get_acl_rule_ordering`, `list_traffic_matching_lists`, `list_wifi_broadcasts`,
`list_networks`, `list_firewall_zones`, `list_dns_policies`, `list_vpn_tunnels`,
`list_vpn_servers`, `get_device_stats`, `list_pending_devices`, `list_vouchers`,
`list_radius_profiles`.

If destructive tools are enabled (`UNIFI_ALLOW_DESTRUCTIVE=true`), note that this
audit may surface items you want to remediate in the same session. The optional
`delete_voucher` tool is used in Section 8 only when destructive tools are enabled.

---

## Section 1 — Device inventory and rogue device detection

**Goal:** Identify every device on the network and flag anything unexpected.

1. Call `list_devices` (paginate until all pages retrieved). For each device note:
   - `name`, `model`, `macAddress`, `ipAddress`, `state`, `firmwareVersion`, `firmwareUpdatable`

2. Call `list_pending_devices`. Any device visible on the network but **not yet
   adopted** is a potential rogue or misconfigured device. Flag all results as
   **[HIGH]** findings — an unadopted device has unrestricted access to the network
   and its origin is unverified.

3. Call `list_clients` (paginate until complete). For each connected client note:
   - `name`, `macAddress`, `ipAddress`, `type`, `uplinkDeviceId`, `connectedAt`
   - Flag clients with **no name** (`name` is empty) — unrecognised clients are
     worth investigating.
   - Flag any client whose `connectedAt` timestamp is unusually recent if the
     owner was not expecting new devices.

**Findings to raise:**
- `[HIGH]` Pending (unadopted) devices present on the network
- `[MEDIUM]` Unnamed or unrecognised connected clients
- `[INFO]` Total device and client counts for the record

---

## Section 2 — Firmware currency

**Goal:** Ensure all adopted devices are running current firmware to eliminate
known CVEs.

1. From the `list_devices` results collected in Section 1, check each device's
   `firmwareUpdatable` field.
2. Call `get_device_stats` for any device that appears unhealthy (high CPU, high
   memory, or long uptime without reboots may indicate a hung process).

**Findings to raise:**
- `[HIGH]` Any device with `firmwareUpdatable: true` — outdated firmware is the
  most common attack surface on home networks. List device name, model, and
  current version.
- `[MEDIUM]` Any device with CPU utilisation > 80% or memory utilisation > 90%
  sustained — may indicate a compromised process or misconfiguration.

---

## Section 3 — WiFi configuration

**Goal:** Verify SSIDs use strong encryption and guest networks are isolated.

1. Call `list_wifi_broadcasts` (paginate until complete). For each broadcast note:
   - `name` (SSID), `enabled`, `security` (or equivalent encryption field),
     `guestPolicy` / `isGuest`, `hiddenSsid`

**Check for:**
- Any SSID with **open (no encryption)** — flag `[CRITICAL]`
- Any SSID with **WEP or WPA (TKIP)** — flag `[HIGH]`; only WPA2-AES or WPA3 are
  acceptable
- Any SSID marked as guest that does **not** have client isolation or a guest
  policy — flag `[HIGH]`
- Any **non-guest SSID** shared with IoT or untrusted devices — flag `[MEDIUM]`;
  recommend a dedicated IoT SSID on a separate VLAN
- SSIDs that are **enabled but not in use** (no associated clients) — flag
  `[LOW]`; unnecessary broadcast surface

**Findings to raise:**
- `[CRITICAL]` Open or WEP-encrypted SSIDs
- `[HIGH]` WPA-TKIP SSIDs; guest SSIDs without isolation
- `[MEDIUM]` IoT devices on primary SSID
- `[LOW]` Enabled but unused SSIDs

---

## Section 4 — Network segmentation

**Goal:** Confirm VLANs and zones provide meaningful separation between trust
boundaries (LAN, IoT, guest, management).

1. Call `list_networks`. For each network note the name, purpose (inferred from
   name), and VLAN ID if present.
2. Call `list_firewall_zones`. For each zone note the name and associated
   `networkIds`.

**Check for:**
- All traffic in a single flat network (no VLANs) — flag `[MEDIUM]`; recommend
  at minimum: LAN, IoT, Guest
- Firewall zones defined but no policies between them — flag `[MEDIUM]` (zones
  without policies provide no actual isolation)
- IoT devices sharing a zone with trusted LAN devices — flag `[MEDIUM]`

**Findings to raise:**
- `[MEDIUM]` Flat network with no VLAN separation
- `[MEDIUM]` Zones defined but not enforced by policies

---

## Section 5 — Firewall policy and ACL review

**Goal:** Ensure firewall rules follow least-privilege and have no permissive
rules that undermine segmentation.

1. Call `list_firewall_policies` (paginate until complete). For each policy note:
   - `name`, `enabled`, `action` (ALLOW/BLOCK), source zone, destination zone,
     description if present

2. Call `list_acl_rules` and `get_acl_rule_ordering`. Note the evaluation order
   — rules are evaluated top-down and the first match wins.

3. Call `list_traffic_matching_lists` to understand what IP/port sets the
   policies reference.

**Check for:**
- Any `ALLOW` policy from a **less trusted zone to a more trusted zone** with no
  specific port/protocol constraint (i.e. allow-all from IoT → LAN) — flag
  `[HIGH]`
- Any policy or ACL rule that is **disabled** — flag `[LOW]`; disabled rules are
  often forgotten; confirm they are intentionally inactive
- Any `ALLOW` rule that **shadows** a later `BLOCK` rule (broad ALLOW before
  specific BLOCK) — flag `[HIGH]`
- Firewall policies for zones that **no longer exist** — flag `[LOW]`; stale
  rules are clutter and potential confusion
- `ALLOW` rules with empty or `any` source/destination where specificity was
  intended — flag `[MEDIUM]`

**Findings to raise:**
- `[HIGH]` Over-permissive cross-zone ALLOW rules; shadowed BLOCK rules
- `[MEDIUM]` Overly broad source/destination on ALLOW rules
- `[LOW]` Disabled rules; stale rules referencing deleted zones

---

## Section 6 — DNS policy anomalies

**Goal:** Detect unexpected local DNS overrides that could redirect traffic.

1. Call `list_dns_policies` (paginate until complete). For each policy note:
   - `domain`, `ipv4Address`, `type`, `enabled`

**Check for:**
- Any policy overriding a **well-known public domain** (e.g. `google.com`,
  `github.com`, a bank domain) to an internal IP — flag `[HIGH]`; this is a
  classic DNS hijack pattern
- Any policy pointing to an **unexpected or unrecognised IP** — flag `[MEDIUM]`
- Disabled policies — flag `[INFO]`; confirm they are intentionally inactive

**Findings to raise:**
- `[HIGH]` Public domain overridden to an internal/unexpected IP
- `[MEDIUM]` DNS policy pointing to an unrecognised IP address

---

## Section 7 — VPN configuration review

**Goal:** Ensure VPN tunnels and servers are correctly configured and no
unexpected tunnels exist.

1. Call `list_vpn_tunnels`. For each tunnel note the name, status, and remote
   endpoint.
2. Call `list_vpn_servers`. For each server note the name, enabled state, and
   protocol.

**Check for:**
- Any tunnel or server the owner does **not recognise** — flag `[HIGH]`
- VPN servers that are **enabled but not in use** — flag `[LOW]`; unnecessary
  attack surface
- VPN tunnels with **no description/name** — flag `[INFO]`; undocumented tunnels
  are hard to audit

**Findings to raise:**
- `[HIGH]` Unrecognised VPN tunnels or servers
- `[LOW]` Enabled but unused VPN servers

---

## Section 8 — Hotspot voucher hygiene

**Goal:** Ensure no stale vouchers grant persistent or unintended network access.

1. Call `list_vouchers` (paginate until complete). Note any vouchers with:
   - No expiry (`time_limit_minutes: 0`) — unlimited time vouchers are risky
   - No data cap (`data_limit_mb: 0`) — unlimited data vouchers can be abused
   - No bandwidth limit — unlimited bandwidth vouchers can saturate the WAN

**Findings to raise:**
- `[MEDIUM]` Vouchers with unlimited time and no data/bandwidth cap
- `[LOW]` Large number of unused vouchers — if destructive tools are enabled (`UNIFI_ALLOW_DESTRUCTIVE=true`) and you have appropriate authorization, consider revoking them with `delete_voucher`

---

## Section 9 — RADIUS profile review

**Goal:** Confirm RADIUS profiles are intentional and do not expose authentication
services unnecessarily.

1. Call `list_radius_profiles` (paginate until complete). For each profile note the
   name, enabled state, and whether it is assigned to any SSID or network.

**Check for:**
- Any profile the owner does **not recognise** — flag `[HIGH]`
- Profiles that are **enabled but not assigned** to any SSID or network — flag
  `[LOW]`; unused RADIUS profiles are unnecessary configuration that can be confused
  for active auth infrastructure
- Profiles using shared secrets that are not rotated periodically — flag `[INFO]`
  as a reminder to rotate

**Findings to raise:**
- `[HIGH]` Unrecognised RADIUS profiles
- `[LOW]` Enabled RADIUS profiles not assigned to any SSID or network

---

## Section 10 — Security limitations of unifi-mcp

The unifi-mcp tools expose the **configuration and inventory** APIs. Some
security-relevant data is **not available** via these tools and requires direct
access to the UniFi console UI or syslog:

| Data type | Where to find it |
|---|---|
| IDS/IPS threat alerts | UniFi UI → Security → Threat Management |
| Traffic anomaly alerts | UniFi UI → Security → Traffic Anomalies |
| Client block/connection events | UniFi UI → Clients → select client → History |
| System event log | UniFi UI → System Log |
| WAN firewall hit counters | UniFi UI → Firewall Policies → expand rule |
| DDoS / port scan detection | UniFi UI → Security → DDoS Protection |

After completing this automated audit, manually review the above sections in the
UniFi UI to complete the security picture.

---

## Output — Prioritised Findings Report

After completing all sections, produce a report in this format:

```
# Network Security Audit — <date>

## Summary
- Devices audited: N adopted, M pending
- Clients connected: N
- Total findings: X critical, Y high, Z medium, W low, V info

## Findings

### [CRITICAL] <title>
**Section:** <N>
**Detail:** <what was found, specific names/IPs/IDs>
**Recommendation:** <what to do>

### [HIGH] <title>
...

### [MEDIUM] <title>
...

### [LOW] <title>
...

### [INFO] <title>
...

## Items requiring manual review in UniFi UI
- [ ] IDS/IPS threat alerts
- [ ] Traffic anomaly alerts
- [ ] System event log
- [ ] WAN firewall hit counters
```

Severity definitions:
- **CRITICAL** — active or near-certain risk; remediate immediately
- **HIGH** — significant exposure; remediate before next audit
- **MEDIUM** — meaningful risk; remediate within a reasonable window
- **LOW** — hygiene issue or unnecessary attack surface; remediate when convenient
- **INFO** — noted for the record; no action required
