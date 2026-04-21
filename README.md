# Splunk + Sysmon Integration — Troubleshooting & Configuration

**Homelab portfolio project — Arturs Kaufmanis | ADForest.local | April 2026**

---
(https://arturskaufmanis.github.io/Sysmon-Splunk-Integration/splunk-sysmon-troubleshooting.html)
## What This Is

This project documents the troubleshooting and resolution of a Sysmon sourcetype misassignment in a Splunk Enterprise homelab environment. Sysmon events were ingesting correctly but landing under the generic `xmlwineventlog` sourcetype instead of the dedicated `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`, breaking targeted detection queries and dashboard reliability.

The writeup follows the full diagnostic chain — from initial evidence gathering through btool inspection to identifying the root cause inside a vendor TA — and demonstrates real-world Splunk administration methodology.

> **Note:** This is a follow-on project. The initial Splunk + Sysmon setup (installation, TA deployment, Universal Forwarder configuration) is documented separately in [`splunk-sysmon-homelab`](../splunk-sysmon-homelab).

---

## Environment

| Component | Details |
|---|---|
| SIEM | Splunk Enterprise 10.2 (Free Licence — 500 MB/day) |
| Host | WIN-ESVD1CAD1FJ.ADForest.local (Windows Server 2025) |
| Sysmon | Microsoft Sysmon64 (Active) |
| Splunk TA | Splunk_TA_microsoft_sysmon (installed) |
| Index | `wineventlog` |
| Target Sourcetype | `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` |

---

## The Problem

Sysmon was running and generating events. Splunk was ingesting them. But:

- `sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"` → **0 results**
- `source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"` → **86,558 events**, all tagged as `xmlwineventlog`

The data existed — it was mislabelled.

---

## Root Cause

The TA's `default/props.conf` contained a backward-compatibility rename directive:

```
[XmlWinEventLog:Microsoft-Windows-Sysmon/Operational]
rename = xmlwineventlog
```

The `rename` directive operates above Splunk's standard configuration merge order and overrides `inputs.conf` sourcetype declarations regardless of file location. `btool` confirmed the `inputs.conf` setting was correct — the TA was silently overriding it.

---

## Resolution

Created a `local/` directory override within the TA (correct industry practice — never modify `default/` directly):

**`Splunk_TA_microsoft_sysmon\local\props.conf`**
```ini
[source::XmlWinEventLog:Microsoft-Windows-Sysmon/Operational]
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

[XmlWinEventLog:Microsoft-Windows-Sysmon/Operational]
rename =
```

The empty `rename =` cancels the TA default. The `source::` stanza ensures correct tagging at ingestion.

A secondary issue was also fixed: Event ID 1 (Process Create) had been placed on both the whitelist and blacklist in `inputs.conf`. Splunk's blacklist takes precedence, so it was silently dropped. Removing it from the blacklist restored collection.

---

## Key Skills Demonstrated

- **Splunk configuration precedence** — understanding how `rename` in `props.conf` overrides `inputs.conf` regardless of file location
- **btool diagnostic methodology** — using `splunk btool` to inspect merged configuration and isolate the override source
- **TA customisation best practice** — `local/` directory overrides that survive TA updates
- **WinEventLog whitelist/blacklist interaction** — blacklist takes precedence; conflicting entries cause silent data loss
- **Systematic troubleshooting** — confirm data exists → locate it → verify each config layer → trace the override

---

## Files

| File | Description |
|---|---|
| `splunk-sysmon-troubleshooting.html` | Full portfolio writeup — rendered documentation |

---

## Related Project

The initial setup project (Sysmon installation, TA deployment, Universal Forwarder configuration) is documented in a separate repository:

→ [`splunk-sysmon-homelab`](../splunk-sysmon-homelab)

---

*Arturs Kaufmanis — [github.com/arturskaufmanis](https://github.com/arturskaufmanis)*
