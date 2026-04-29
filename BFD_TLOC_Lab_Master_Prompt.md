# MASTER PROMPT — BFD TLOC Up/Down Viptela Ansible Automation Package
# For Lab-Testable Rebuild

---

## WHO YOU ARE

You are a senior Ansible automation engineer building production-grade network automation
playbooks for Cisco SD-WAN (Viptela) environments. You write clean, minimal, no-comment
YAML and Python. Every piece of code you write must be immediately testable in a real lab
using real Cisco SD-WAN devices (vEdge and cEdge/IOS-XE).

---

## WHAT THIS PACKAGE IS

Package name      : BFD_TLOC_Up_Down_Viptela
Alarm type        : BFD TLOC Up/Down on Cisco SD-WAN Viptela devices
Device platforms  : Cisco vEdge (native OS) and Cisco cEdge (IOS-XE with SD-WAN)
Alarm source      : Assure1 monitoring tool (sourced into ETMS ticketing)
Target platform   : Verizon vCAT automation (Jenkins-based, Ansible under the hood)
Closure mechanism : Write the string `ava-TRIAGE-DeviceUpDown` to output.log — ETMS
                    scans for this and auto-closes the ticket. No API call needed.

---

## THE ALARM TEXT FORMAT

When this alarm fires, the ETMS ticket contains a TEXT field in this exact format:

    TEXT: All BFD sessions for the tloc are down- public-internet spdway-sdstr9956-30106801e001 10009956 2.1.99.56

Breaking it down:
- `public-internet` = TLOC color (could be biz-internet, mpls, lte, etc.)
- `spdway-sdstr9956-30106801e001` = device hostname
- `10009956` = system IP suffix / site ID
- `2.1.99.56` = IP

The TLOC color must be extracted from this string using regex: `r'down-\s+(\S+)'`

In the vCAT environment, this text is available inside the variable called `Problem Summary`
(space in the key name — access in Jinja2 as `{{ vars['Problem Summary'] }}`).
The full Problem Summary contains multiple fields including TEXT. Extract TEXT first,
then extract color from TEXT.

FOR LAB TESTING: hardcode the color as a variable at the top of site.yml:
    tloc_color: "public-internet"
And also hardcode:
    detected_platform: "vedge"   # or "ios_xe" depending on your lab device
This replaces the ETMS Problem Summary parsing for lab runs.

---

## INVENTORY / HOW THE PLAYBOOK IS TRIGGERED

In production (vCAT): ETMS auto-generates a JSON inventory file and passes it to Ansible.
The JSON structure is:

```json
{
  "all": {
    "hosts": {
      "<hostname>": {
        "MgmtIP": "10.x.x.x",
        "ansible_host": "10.x.x.x",
        "ansible_user": "**** TACACS_USER_NAME ****",
        "ansible_password": "**** TACACS_USER_PWD ****",
        "ansible_network_os": "ios",
        "ansible_connection": "network_cli",
        "ansible_port": 22,
        "ansible_command_timeout": 180,
        "ansible_become": false,
        "ansible_python_interpreter": "python",
        "Problem Summary": "INCIDENT START TIME: ...\nNEID: ...\nTEXT: All BFD sessions for the tloc are down- public-internet ...",
        "request": {
          "requestId": "VCATS120824",
          "requestTimestamp": "04-03-2026 09:23:09 GMT"
        },
        "credentials": {
          "cinfo_1": [{"username": "...", "password": "..."}],
          "cinfo_4": [{"username": "...", "password": "..."}]
        }
      }
    },
    "vars": {
      "vcatsAuthToken": ""
    }
  }
}
```

FOR LAB TESTING: Create a simple `lab_inventory.yml`:

```yaml
all:
  hosts:
    lab-vedge-01:
      ansible_host: 192.168.1.10        # your lab device IP
      ansible_user: admin
      ansible_password: "yourpassword"
      ansible_network_os: ios
      ansible_connection: network_cli
      ansible_port: 22
      ansible_command_timeout: 180
      ansible_become: false
      MgmtIP: "192.168.1.10"
      tloc_color: "public-internet"     # lab override
      detected_platform: "vedge"        # lab override: vedge or ios_xe
```

NEVER hardcode credentials in playbooks. For lab, put them in the inventory file only.

---

## PACKAGE STRUCTURE (STRICT — DO NOT ADD OR REMOVE FILES)

```
BFD_TLOC_Up_Down_Viptela/
├── filter_plugins/
│   └── bfd_tloc_filters.py        ← All custom Jinja2 filters
└── playbooks/
    ├── site.yml                   ← Master orchestrator
    ├── 02_platform_detection.yml  ← SSH + platform detect + color extract
    ├── 03_bfd_diagnostics.yml     ← 4 core diagnostic commands + BFD state eval
    ├── 04_control_tloc_check.yml  ← Control plane check + BFD flap logs
    ├── 05_interface_remediation.yml ← TLOC local/remote + interface bounce
    ├── 06_redundant_router_check.yml ← Redundant router SSH + BFD check
    ├── 07_omp_ipsec_check.yml     ← OMP + IPsec + single peer underlay check
    └── 08_alarm_clearance.yml     ← Final BFD recheck + alarm clear stubs + closure
```

No ansible.cfg. No roles/. No group_vars/. No requirements.txt. No hosts.ini.
No vault files. No collections/.

---

## THE FULL MOP FLOW (HTML DIAGRAM — EVERY NODE EXTRACTED)

This is the approved MOP. Every block must be implemented. Read carefully.

### BLOCK 1 — Ticket Ingestion (NOT CODED — handled by vCAT/ETMS externally)
- Incident received from ETMS/VBMS
- Extract ticket data: Hostname, NEID, IP, Alert time, alarm detail
- TLOC color is embedded in the TEXT field of the alarm

### BLOCK 2 — SSH + Platform Detection (Phase 02)
- Login to device (IP from ticket) via SSH
- Run `show version`
  - vEdge: also run `show system status`
  - cEdge: also run `show sdwan system status`
- Identify: is this a vEdge or cEdge?
  - vEdge detected by: 'vEdge' or 'vedge' or 'SD-WAN' in show version output
  - cEdge detected by: 'IOS-XE' or 'IOS XE' in show version output
  - On vEdge, show version returns SHORT output (e.g. just "20.6.4")
  - On cEdge, show version returns LONG multi-line IOS-XE output
- For cEdge: ALL SD-WAN commands start with `show sdwan`
- Extract TLOC color from Problem Summary TEXT field

### BLOCK 3 — Check if Control TLOC Down ticket exists in ETMS (Phase 02, continued)
- Check ETMS for an existing Control TLOC Down ticket for the same device
- LAB NOTE: This is an ETMS step. For lab, SKIP this ETMS query entirely.
  Just print a message: "ACTION REQUIRED: Manually check ETMS for existing
  Control TLOC Down ticket for {{ inventory_hostname }} before proceeding."
  Write this message to output.log and continue — do NOT stop.

### BLOCK 4 — Run BFD Diagnostic Commands (Phase 03)
Run ALL of the following. Both platforms branch separately:

vEdge commands:
- show version
- show hardware inventory
- show bfd sessions local-color <color>
- show bfd history color <color>

cEdge commands:
- show sdwan version
- show inventory
- show sdwan bfd sessions | include <color>|COLOR
- show sdwan bfd history | include <color>|COLOR

Parse the BFD sessions output. Evaluate:
- Are ALL sessions for this color UP?
- Are ANY sessions for this color DOWN?

### BLOCK 5 — BFD Session Up Path (Phase 08 — runs only if all BFD up)
- If BFD is up and stable for > 1 hour (check uptime in BFD session table)
  LAB: Check BFD uptime from parsed session output. Threshold = 15 minutes minimum.
- Check vManage for stuck alarm
  LAB/UNKNOWN: Leave as STUB — print "STUB: vManage alarm check would run here"
  In production: call vmanage_module_path script
- Check Assure1 for stuck alarm
  LAB/UNKNOWN: Leave as STUB — print "STUB: Assure1 alarm check would run here"
  In production: call assure1_module_path script
- If all clear: Update logs, assign to L1 NOC for ticket closure
  Write `ava-TRIAGE-DeviceUpDown` to output.log

### BLOCK 6 — BFD Down Path: Check Control TLOC (Phase 04)
- Run show control connections / show sdwan control connections
- Is Control TLOC down?
  - YES: Print "Control TLOC is DOWN. Refer to Control TLOC Down MOP."
          Collect commands below and stop:
          show sdwan control connections / show control connections
          (These are evidence for the escalation — write to output.log)
  - NO (Control TLOC is UP): Continue to Block 7

### BLOCK 7 — BFD Down, Control Up: Check Flapping (Phase 04, continued)
Run to check if BFD is flapping:
- show bfd sessions (vEdge) / show sdwan bfd sessions (cEdge)
- show bfd sessions detail (vEdge) / show sdwan bfd sessions detail (cEdge)
- show log | i BFD
- show log | i tloc

Is BFD TLOC flapping? (check uptime and transitions columns — frequent transitions = flapping)
- YES: Write outputs to log. Print "Assign Tier-2 team to investigate BFD flapping on <device>"
       Stop.
- NO: Continue to Block 8

### BLOCK 8 — Interface and TLOC Termination Check (Phase 05)
If control TLOC is up:
- Run: show interface <interface-name> / show sdwan interface <interface-name>
- Run: show interface description
- Run: show sdwan control local-properties / show control local-properties

Check: Is TLOC terminated on THIS device?
Method: Compare public IP and private IP from show control local-properties output
- If public IP == private IP → TLOC terminates HERE → go to Block 9 (interface bounce)
- If public IP != private IP → TLOC terminates on REDUNDANT router → go to Block 10

Also: If interface description contains "over TLOC extension" → TLOC is on redundant router

### BLOCK 9 — TLOC Local: Check Template + Bounce Interface (Phase 05, continued)
- Run: show sdwan system status / show system status
- Is device managed by a template group?

  NO (not template managed):
    - Bounce interface from CLI:
        interface <interface-name>
         shutdown
         no shutdown
    - Wait 30 seconds
    - Re-check interface: show interface / show sdwan interface
    - Interface up? → Continue to Block 11 (BFD re-check)
    - Interface still down? → Print "Assign field engineer to check physical connectivity"
                              Stop.

  YES (template managed):
    - LAB NOTE: vManage template API bounce. For lab, this is a STUB.
      Print "STUB: vManage template bounce would be triggered via API here"
      Print "Manual action: Login to vManage → Monitor → Network → find device →
             edit template → shut/no-shut the {{ tloc_color }} interface"
      In production: POST to /dataservice/template/device/config/attachfeature
    - Wait 30 seconds
    - Re-check interface: show interface / show sdwan interface
    - Interface up? → Continue to Block 11 (BFD re-check)
    - Interface still down? → Print "Assign field engineer to check physical connectivity"
                              Stop.

### BLOCK 10 — TLOC on Redundant Router (Phase 06)
- TLOC is NOT terminated on this device. Must check the redundant router.
- How to find redundant router:
  - Run show vrrp brief and show vrrp detail on primary
  - Run show interface description
  - Run show ip interface brief
  - Run show vrf (check if VRF is applied)
  - Derive peer IP using odd/even last-octet rule:
      If primary last octet is ODD → peer is EVEN (next IP)
      If primary last octet is EVEN → peer is ODD (previous IP)
  - VRF check: if VRF present on LAN interface → SSH with VRF:
      ssh -l <username> -vrf 1 <peer-ip>
    If no VRF → normal SSH

- Dynamically add redundant router to Ansible in-memory inventory using add_host
- SSH to redundant router
- Run:
    show bfd sessions local-color <color> (vEdge)
    show sdwan bfd sessions | include <color>|COLOR (cEdge)
    show control connections / show sdwan control connections
    show interface description
- Is BFD up on redundant router?
  - YES: Continue — check back-to-back connectivity
         Run: show interface <interface-name> / show sdwan interface <interface-name>
         Interface up? YES → Print "Assign Tier-2 team to investigate" / Stop
         Interface up? NO  → Print "Assign Tier-2 team to check back-to-back connectivity" / Stop
  - NO: Print "Assign Tier-2 team to check back-to-back connectivity between routers"
        Stop.

### BLOCK 11 — BFD Re-check After Bounce (Phase 08, final re-check path)
After interface bounce (from Block 9):
- Run:
    show bfd sessions / show bfd summary (vEdge)
    show sdwan bfd sessions / show sdwan bfd history (cEdge)
- Is BFD TLOC now up?
  - YES: Uptime check → alarm clear → closure (same as Block 5 happy path)
  - NO:  Print "Assign Tier-2 team to investigate — BFD still down after interface bounce"
         Stop.

### BLOCK 12 — Single Peer Down Path (Phase 07)
If BFD is down for only ONE peer (not all peers):
- Check underlay reachability to that peer:
    ping <dst-ip> source <src-ip>
  (Get dst-ip and src-ip from: show sdwan bfd sessions table)
- Ping successful?
  - NO: LAB NOTE: ETMS cross-reference is a stub for lab.
        Print "ACTION REQUIRED: Check ETMS for existing ticket for remote device <peer-ip>"
        Print "Suspected remote device/circuit/ISP issue."
        Stop.
  - YES: Check tunnel to specific peer:
         show sdwan tunnel statistics | include <peer-system-ip>
         show sdwan tunnel statistics table | i <system-ip> (check TX/RX packets)
         vEdge: show tunnel connections

         Tunnel down?
         - YES: Print "Assign Tier-2 team to check NAT/Firewall on path to <peer>"
                Stop.
         - NO (tunnel up, BFD still down):
                Check TLOC config match:
                  Verify encapsulation (ipsec/gre), color, port
                Check localized policy:
                  show sdwan policy data-policy-filter
                Check peer device health:
                  show sdwan system status
                Print "Assign Tier-2 team for further investigation"
                Stop.

### BLOCK 13 — All Peers Down Path (Phase 07)
If BFD is down for ALL peers:
- Check OMP sessions and TLOC routes:
    vEdge: show omp peers / show omp tlocs
    cEdge: show sdwan omp peers / show sdwan omp tlocs
- Are OMP sessions up and required TLOCs learned?
  - NO: Print "OMP/TLOC learning issue — Assign Tier-2 (policy/template review)"
        Stop.
  - YES: Check data-plane/IPsec tunnels:
           vEdge: show tunnel connections / show bfd sessions
           cEdge: show sdwan tunnel statistics / show sdwan ipsec sa /
                  show sdwan tunnels / show sdwan bfd sessions
         Are IPsec tunnels formed for this TLOC?
         - NO: Print "Assign Tier-2 to verify: NTP sync, NAT traversal,
                      ACL/Firewall blocking UDP 12346/12366, MTU/PMTUD"
               Stop.
         - YES (tunnels up, BFD still down):
               Print "Persisting BFD Down with Control TLOC Up —
                      Assign Tier-2 with collected outputs"
               Stop.

---

## EVIDENCE BUNDLE (collected at every Tier-2 escalation point)

The following commands appear as the evidence collection set in the MOP.
Run ALL of them before every Tier-2 escalation and write to output.log:

vEdge                                      cEdge
show version                               show sdwan version / show version
show hardware inventory                    show inventory
show bfd sessions <local-color> color      show sdwan bfd sessions | include <color>|COLOR
show bfd history color <color>             show sdwan bfd history | include <color>|COLOR
show control connections                   show sdwan control connections
show omp peers                             show sdwan omp peers
show omp tlocs                             show sdwan omp tlocs
show interface                             show sdwan interface
show tunnel statistics                     show sdwan tunnel statistics
show system statistics                     show system statistics
show processes                             show processes cpu
show ip route                              show ip route
show policy service-path                   show sdwan policy service-path

---

## OUTPUT.LOG FORMAT

Every command output must be written to output.log in this exact format:

```
==================== SUMMARY REPORT ====================
Hostname / IP     : {{ inventory_hostname }} / {{ MgmtIP }}
NEID              : {{ inventory_hostname }}
Request ID        : {{ request.requestId | default('LAB-TEST') }}
Timestamp         : {{ request.requestTimestamp | default('N/A') }}
--------------------------------------------------------
=++=++=++=++=++=++=++=++=++=++=++=++=++=++=
<command name here>
------------------------------------------
<command output here>
=++=++=++=++=++=++=++=++=++=++=++=++=++=++=
```

Closure tag (written at very bottom when incident is auto-closed):
```
ava-TRIAGE-DeviceUpDown
```

---

## PLAYBOOK RULES (NON-NEGOTIABLE)

- hosts: all
- gather_facts: false
- Task naming: "NNNN | PHASE | Description" e.g. "0301 | BFD | Run show version"
- No inline comments in YAML task blocks
- No hardcoded IPs, usernames, passwords, or tokens anywhere in playbooks
- All device commands must be platform-branched (vEdge vs cEdge)
- Every playbook appends its outputs to output.log
- Use `delegate_to: localhost` for all file write tasks
- Use `ansible.builtin.blockinfile` with `marker: ""` for appending to output.log
- Use `cisco.ios.ios_command` for all device CLI commands
- Use `cisco.ios.ios_config` for any configuration changes (interface bounce)
- Use `ansible.builtin.set_fact` to persist data between tasks
- Cross-play variable access: `hostvars[inventory_hostname]['var_name']`
- Cross-play to second host: `hostvars[groups['all'][0]]['var_name']`
- Gate every phase with: skip if `bfd_all_up` is True (phases 04-07)

---

## FILTER PLUGIN RULES

File: `filter_plugins/bfd_tloc_filters.py`
All custom Jinja2 filters go here. No dead code.

Required filters:

```
detect_platform(show_version_output)
  → 'vedge', 'ios_xe', or 'unknown'
  → vEdge: 'vEdge' or 'vedge' or 'SD-WAN' in output
  → cEdge: 'IOS-XE' or 'IOS XE' in output

extract_text_field(problem_summary)
  → regex r'TEXT:\s*(.+)' on full Problem Summary string

extract_tloc_color(text_field)
  → regex r'down-\s+(\S+)' on TEXT field value

parse_bfd_sessions_vedge(output, tloc_color)
  → list of dicts: system_ip, site_id, state, src_color, uptime, transitions
  → filter rows to only those matching tloc_color

parse_bfd_sessions_cedge(output, tloc_color)
  → same structure as vedge version

all_bfd_sessions_up(sessions)
  → True if all session['state'] == 'up'

any_bfd_session_down(sessions)
  → True if any session['state'] != 'up'

parse_uptime_seconds(uptime_str)
  → handles '0:18:01', '1h30m', '2d3h' → integer seconds

bfd_uptime_above_threshold(sessions, threshold_minutes)
  → True if all sessions uptime > threshold_minutes * 60

parse_bfd_history(output, tloc_color)
  → list of dicts: system_ip, site_id, color, state, raw

parse_control_connections(output)
  → list of dicts: peer_ip, type, state, raw

control_tloc_up(connections)
  → True if any connection state == 'up'

parse_local_properties(output)
  → dict: {public_ip, private_ip}

tloc_terminated_locally(local_properties)
  → True if public_ip == private_ip

parse_omp_peers(output)
  → list of dicts: peer, type, state, raw

omp_sessions_up(peers)
  → True if any peer state == 'up'

parse_interface_status(output, interface_name)
  → 'up', 'down', 'admin-down', 'unknown'

parse_template_status(output)
  → True if 'attached' or 'true' or 'enabled' in template line

parse_ipsec_tunnels(output, tloc_color)
  → list of tunnel entries matching tloc_color

parse_vrrp_output(output)
  → dict: {vrrp_state, primary_ip, virtual_ip}

derive_peer_ip(mgmt_ip)
  → if last octet is odd → increment by 1
  → if last octet is even → decrement by 1

parse_ping_result(output)
  → True if '0% packet loss' or 'Success rate is 100' in output
  → False otherwise

detect_bfd_flapping(sessions)
  → True if any session has transitions > 5 (high churn)
```

---

## PLATFORM BRANCHING PATTERN (use in every playbook)

```yaml
vars:
  platform: "{{ hostvars[inventory_hostname]['detected_platform'] }}"
  color: "{{ hostvars[inventory_hostname]['tloc_color'] }}"

tasks:
  - name: "NNNN | PHASE | Run command (vEdge)"
    cisco.ios.ios_command:
      commands:
        - show bfd sessions local-color {{ color }}
    register: result_vedge
    when: platform == 'vedge'

  - name: "NNNN | PHASE | Run command (cEdge)"
    cisco.ios.ios_command:
      commands:
        - "show sdwan bfd sessions | include {{ color }}|COLOR"
    register: result_cedge
    when: platform == 'ios_xe'

  - name: "NNNN | PHASE | Set unified fact (vEdge)"
    ansible.builtin.set_fact:
      result_output: "{{ result_vedge.stdout[0] }}"
    when: platform == 'vedge'

  - name: "NNNN | PHASE | Set unified fact (cEdge)"
    ansible.builtin.set_fact:
      result_output: "{{ result_cedge.stdout[0] }}"
    when: platform == 'ios_xe'
```

---

## ESCALATION PATTERN (use at every Tier-2 / field engineer exit point)

```yaml
  - name: "NNNN | PHASE | Write escalation to output.log"
    ansible.builtin.blockinfile:
      path: "{{ playbook_dir }}/../output.log"
      marker: ""
      block: |
        =++=++=++=++=++=++=++=++=++=++=++=++=++=++=
        ESCALATION — TIER-2 REQUIRED
        ------------------------------------------
        Reason: <specific reason>
        Device: {{ inventory_hostname }}
        TLOC Color: {{ color }}
        Action: Assign Tier-2 team to check <specific thing>
        =++=++=++=++=++=++=++=++=++=++=++=++=++=++=
    delegate_to: localhost

  - name: "NNNN | PHASE | Print escalation message"
    ansible.builtin.debug:
      msg: "Assign Tier-2 team to check <specific reason> on {{ inventory_hostname }}"

  - name: "NNNN | PHASE | End play"
    ansible.builtin.meta: end_play
```

---

## STUBS — WHAT TO LEAVE BLANK FOR LAB

These items are unknown or untestable in lab. Write them as stubs that print clearly
and do not break execution:

### 1. ETMS Cross-Reference (Control TLOC Down ticket check)
Do NOT call any ETMS API. Just print:
```yaml
  - name: "NNNN | CONTROL | Print ETMS cross-reference instruction"
    ansible.builtin.debug:
      msg: >
        ACTION REQUIRED: Manually check ETMS for an existing Control TLOC Down ticket
        for device {{ inventory_hostname }}. If one exists, link it to this ticket
        before proceeding. This check cannot be automated without ETMS API endpoints.
```

### 2. ETMS Remote Device Ticket Check (single peer ping fails)
Same pattern — print instruction, do not stop execution.

### 3. vManage Alarm Clear
```yaml
  - name: "NNNN | ALARM | STUB — vManage alarm clear"
    ansible.builtin.debug:
      msg: >
        STUB: vManage alarm clear would run here.
        In production: call {{ vmanage_module_path }} --device {{ inventory_hostname }}
        For lab: manually clear alarm in vManage UI if present.
```

### 4. Assure1 Alarm Clear
```yaml
  - name: "NNNN | ALARM | STUB — Assure1 alarm clear"
    ansible.builtin.debug:
      msg: >
        STUB: Assure1 alarm clear would run here.
        In production: call {{ assure1_module_path }} --device {{ inventory_hostname }}
        For lab: manually clear alarm in Assure1 if present.
```

### 5. vManage Template-Based Interface Bounce
```yaml
  - name: "NNNN | IFACE | STUB — vManage template bounce"
    ansible.builtin.debug:
      msg: >
        STUB: vManage template interface bounce would run here via REST API.
        Endpoint: POST /dataservice/template/device/config/attachfeature
        In production: uses vcatsAuthToken and vmanage_template_id extra vars.
        For lab: manually bounce interface in vManage UI:
        Monitor → Network → find device → edit template → shut/no-shut {{ color }} interface.
```

---

## LAB TESTING INSTRUCTIONS

### Minimum lab setup needed:
- 1x Cisco vEdge device (or cEdge/IOS-XE with SD-WAN) reachable via SSH
- SSH credentials (local or TACACS)
- At least one BFD session configured on the device
- Ansible controller machine with cisco.ios collection installed

### Install required Ansible collection:
```bash
ansible-galaxy collection install cisco.ios
```

### Lab inventory file (lab_inventory.yml):
```yaml
all:
  hosts:
    lab-vedge-01:
      ansible_host: 192.168.1.10
      ansible_user: admin
      ansible_password: "yourpassword"
      ansible_network_os: ios
      ansible_connection: network_cli
      ansible_port: 22
      ansible_command_timeout: 180
      ansible_become: false
      MgmtIP: "192.168.1.10"
      tloc_color: "public-internet"
      detected_platform: "vedge"
      request:
        requestId: "LAB-TEST-001"
        requestTimestamp: "2026-04-16 10:00:00 GMT"
```

### Run command for lab:
```bash
cd BFD_TLOC_Up_Down_Viptela/playbooks
ansible-playbook -i lab_inventory.yml site.yml
```

### For cEdge lab device:
Change in lab_inventory.yml:
```yaml
      ansible_network_os: ios
      detected_platform: "ios_xe"
      tloc_color: "biz-internet"
```

### Check output:
```bash
cat output.log
```

### Verify closure tag was written (happy path):
```bash
grep "ava-TRIAGE-DeviceUpDown" output.log
```

---

## VARIABLE FLOW BETWEEN PLAYBOOKS

Facts set in one playbook persist to all subsequent playbooks in the same
ansible-playbook run. Access them via `hostvars`:

```yaml
# In playbook 03 — set a fact
- name: "Set bfd_all_up"
  ansible.builtin.set_fact:
    bfd_all_up: true

# In playbook 04 — read it back
vars:
  bfd_all_up: "{{ hostvars[inventory_hostname]['bfd_all_up'] }}"
```

For the redundant router play (second play in 06_redundant_router_check.yml)
which runs against a different host, access primary device vars like this:
```yaml
vars:
  primary_platform: "{{ hostvars[groups['all'][0]]['detected_platform'] }}"
  color: "{{ hostvars[groups['all'][0]]['tloc_color'] }}"
```

---

## DECISION GATE PATTERN (every playbook 04–08 starts with this)

```yaml
  - name: "NNNN | PHASE | Skip phase — BFD all sessions up"
    ansible.builtin.meta: end_play
    when: hostvars[inventory_hostname]['bfd_all_up'] | default(false) | bool
```

---

## COMPLETE EXECUTION FLOW SUMMARY

```
[02] SSH → show version → detect vEdge/cEdge → extract TLOC color
     FAIL if color empty or platform unknown
     STUB: print ETMS Control TLOC Down cross-reference instruction

[03] Run 4 diagnostics (version, inventory, bfd sessions, bfd history)
     Parse BFD sessions for TLOC color
     Empty sessions? → Tier-2 escalation → STOP
     Set bfd_all_up, bfd_any_down

[04] bfd_all_up=True? → SKIP to 08
     show control connections → parse → is control TLOC up?
     Control down? → Print "Refer Control TLOC Down MOP" → STOP
     Control up:
       show bfd sessions detail + show log | i BFD + show log | i tloc
       BFD flapping? → Tier-2 escalation → STOP

[05] bfd_all_up=True? → SKIP to 08
     show control local-properties + show interface description
     pub_ip == priv_ip? → TLOC local → continue
     pub_ip != priv_ip? → TLOC on redundant → SKIP to 06
     show interface + show system status
     Template managed?
       NO  → CLI shut/no-shut → wait 30s → re-check interface
       YES → STUB: print vManage template bounce instruction → wait 30s → re-check
     Interface still down? → Field engineer escalation → STOP

[06] bfd_all_up=True? → SKIP
     tloc_local=True? → SKIP
     show vrrp + show ip int brief + show vrf → derive peer IP
     Peer IP unavailable? → Tier-2 escalation → STOP
     add_host: add peer IP as 'redundant_router'
     SSH to redundant router:
       show bfd sessions + show control connections + show interface description
     BFD up on redundant?
       YES → check back-to-back interface → Tier-2 escalation → STOP
       NO  → Tier-2 escalation → STOP

[07] bfd_all_up=True? → SKIP
     Determine: single peer down OR all peers down
     SINGLE PEER:
       ping dst-ip source src-ip
       Ping fails? → STUB ETMS check → print "suspected remote issue" → STOP
       Ping ok?
         check tunnel: show sdwan tunnel statistics | include <peer>
         Tunnel down? → Tier-2 (NAT/Firewall) → STOP
         Tunnel up? → check TLOC config match + policy filter → Tier-2 → STOP
     ALL PEERS:
       show omp peers + show omp tlocs
       OMP down? → Tier-2 (OMP/policy) → STOP
       OMP up:
         show tunnel stats + ipsec sa + tunnels
         No tunnels? → Tier-2 (NTP/NAT/ACL/MTU) → STOP
         Tunnels up, BFD down? → Tier-2 (persisting BFD) → STOP

[08] bfd_all_up=False? → SKIP (already escalated)
     Re-check: show bfd sessions + show bfd history
     BFD still down? → Tier-2 → STOP
     BFD up but uptime < 15min? → Tier-2 (flapping) → STOP
     BFD up, uptime >= 15min:
       STUB: vManage alarm clear debug message
       STUB: Assure1 alarm clear debug message
       Write ava-TRIAGE-DeviceUpDown to output.log
       INCIDENT CLOSED ✅
```

---

## WHAT TO DO AFTER WRITING THE CODE

### Step 1 — Run in lab against a real vEdge or cEdge
```bash
ansible-playbook -i lab_inventory.yml site.yml -v
```

### Step 2 — Test happy path (BFD already up)
Set `bfd_all_up` to force-test the alarm clearance path:
```bash
ansible-playbook -i lab_inventory.yml site.yml -e "force_bfd_up=true"
```

### Step 3 — Check output.log after every test run
```bash
cat ../output.log
```

### Step 4 — Before submitting to vCAT production
Replace the lab_inventory.yml variables with the vCAT ETMS JSON.
Add these runtime extra vars to the vCAT playbook config:
- `vmanage_module_path` — path to vManage alarm clear script on vCAT platform
- `assure1_module_path` — path to Assure1 alarm clear script on vCAT platform
- `vmanage_template_id` — vManage feature template ID for target device type

---

## STYLE RULES (ABSOLUTE)

- Zero inline comments in YAML
- Zero hardcoded credentials, IPs, or tokens in playbooks
- Zero filler or placeholder tasks
- Every task name must be specific and follow NNNN | PHASE | Description
- Python filter functions: short docstring only, no inline comments
- No unused imports in Python
- No dead code in filter plugin
- YAML indentation: 2 spaces everywhere
- String quoting: use double quotes for task names, Jinja2 expressions, and
  any string containing special characters
- All `when:` conditions use `| bool` for boolean facts
- All `ansible.builtin.blockinfile` calls use `marker: ""`
