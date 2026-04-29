# BFD TLOC Up/Down Viptela

## Overview
Automated triage playbook for BFD TLOC Up/Down alarms on Cisco SD-WAN (Viptela) devices.
Triggered by ETMS via vCAT when Assure1 raises a BFDTLOCUpDown incident.

## Supported Devices
- Cisco vEdge (native Viptela OS)
- Cisco cEdge (IOS-XE with SD-WAN)

## Group / Request Type
- Group: ETMS_Triage
- Request Type: BFDTLOCUpDown
- Playbook Name: BFDTLOCUpDown-Cisco-Router

## Credentials Required
- TACACS (device SSH)
- Assure One (alarm validation and clearance)

## What This Playbook Does
1. Parses ETMS Problem Summary to extract TLOC color and Assure1 Alarm ID
2. Detects device platform (vEdge vs cEdge)
3. Cross-references ETMS for existing Control TLOC Down ticket
4. Runs full BFD diagnostic suite
5. Validates control plane, interface state, redundant router if applicable
6. Analyzes peer path for single or all-peers-down scenarios
7. Validates and clears alarms in vManage and Assure1
8. Writes `ava-TRIAGE-DeviceUpDown` to output file only when all 3 conditions pass:
   - BFD stable >= 40 minutes
   - vManage BFD alarm cleared
   - Assure1 ViptelaAlarm cleared

## Possible Outcomes
- **CLOSED**: BFD stable, all alarms cleared — `ava-TRIAGE-DeviceUpDown` written
- **ESCALATION TIER-2**: BFD flapping, tunnel issues, OMP down, persisting BFD down
- **FIELD ENGINEER**: Interface down after bounce
- **DEFERRED**: Existing Control TLOC Down ticket found — defer to Control TLOC Down MOP

## Prerequisites
- Device reachable via SSH on MgmtIP
- vManage reachable from automation runner
- Assure1 credentials available via CINFO
- vcatsAuthToken populated by vCAT at runtime

## Extra Vars (production)
- `vmanage_host`: IP/hostname of vManage instance
- `vmanage_user`: vManage login username
- `vmanage_password`: vManage login password
- `etms_base_url`: ETMS REST base URL (default: https://etms.verizon.com)
- `assure1_auth_b64`: Base64-encoded Assure1 credentials

## Output Files
- `{requestId}_output.txt`: Full evidence bundle and closure tag
