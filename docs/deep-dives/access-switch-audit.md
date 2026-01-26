# Deep Dive: Access Switch Port Audit Tool
### "Enterprise Port Intelligence, Distilled to Pure Python."

The **Access Switch Port Audit Tool** is a production-hardened, multi-threaded intelligence collector designed to map interface health and utilization across your access layer. It transforms raw Cisco CLI telemetry into an executive-ready Excel workbook by combining strategic port classification with real-time operational metrics.

[:material-github: View Source Code on GitHub](https://github.com/Nautomation-Prime/Access_Switch_Audit){ .md-button .md-button--primary }

---

## 🎯 The Nautomation Prime Philosophy in Action

Before diving into the code, understand how every design decision reflects our three core principles:

### **Principle 1: Line-by-Line Transparency**
Every function in this tool includes explicit documentation of *what it does* and *why it's structured this way*. You'll see comments explaining the engineering tradeoffs—why we parse with TextFSM *and* maintain a fallback parser, why we use conservative stale-detection logic, and why conditional formatting in Excel matters for operations teams.

### **Principle 2: Hardened for Production**
Access layer audits run on infrastructure that cannot afford downtime. You'll notice patterns like concurrent connection pooling, per-device failure isolation, graceful fallbacks when commands fail, and secure credential rotation. These aren't "nice to have"—they're mandatory for enterprise reliability.

### **Principle 3: Vendor-Neutral**
This tool is built on industry-standard Python libraries: **Netmiko** (multi-device SSH), **Paramiko** (jump host tunneling), **Pandas & OpenPyXL** (Excel generation), and **TextFSM** (intelligent parsing). Your skills remain portable across vendors.

---

## 🏗️ Technical Architecture

The tool operates as a modular Python application with four primary components:

| Component | Responsibility | Why It Matters |
| :--- | :--- | :--- |
| **CredentialManager** | Secure credential retrieval from OS stores | Passwords never touch plaintext or config files |
| **JumpManager** | Persistent SSH tunneling through a bastion host | Centralizes network access control; supports air-gapped environments |
| **PortAuditor (Netmiko)** | Parallel device SSH connections and command collection | Audits 20+ switches in minutes, not hours |
| **PortIntelligence** | Multi-source port classification and risk flagging | Detects stale, misconfigured, or problematic ports automatically |
| **ExcelReporter** | Professional, templated workbook generation | Operations teams get insights immediately, not raw data dumps |

---

## 🔐 CredentialManager: Secure Credential Handling

### Why Credential Management Matters

**The Problem:** Access layer audits require credentials to 20+ switches. Storing them in plaintext inventory files or hardcoding them in scripts creates unacceptable security debt. Prompting users for each device is error-prone and doesn't scale.

**The Solution:** Integrate with native OS credential stores. Windows has Credential Manager, macOS has Keychain, Linux has pass. These are designed for exactly this use case and provide encryption, access control, and audit trails.

**User Experience:** On first run, the tool prompts for automation credentials (username, password, optional enable secret). Once provided, credentials are stored securely in Windows Credential Manager. On subsequent runs, the tool retrieves them automatically. Operators never type credentials into the tool twice.

### Architecture: Primary + Fallback Credentials

The tool supports a **two-credential fallback model**:

- **Primary:** Your main automation account (typically AD-backed, with broad device permissions)
- **Secondary:** A fallback device account (typically local, with limited scope)

**Why This Design?**
- Jump host always uses primary (tighter access control)
- If primary authentication fails on a device, the tool automatically retries with secondary
- Maximizes success rates in heterogeneous device populations

---

## 🔌 PortAuditor: The Threaded Collection Engine

### Why Parallel Port Auditing is Essential

**The Problem:** Auditing 50 switches serially with 5 commands per device = 250 SSH round-trips. At 2 seconds per connection, that's 8+ minutes of waiting.

**The Solution:** Thread pool with 10 concurrent workers = 10 simultaneous SSH sessions. Same 50 switches audited in 1-2 minutes.

### Thread-Safe Architecture

```python
# Thread-safe accumulators (protected by locks)
self.device_records = []       # Parsed results: one row per device
self.interface_details = []    # Detailed per-interface data
self.failed_devices = {}       # {ip: error_message}
self.progress_lock = threading.Lock()  # Protects shared state
```

**Why Thread Locks Matter:**
- Without locks, multiple threads writing to the same list causes data corruption
- The lock ensures atomic append operations
- Minimal lock contention because we hold locks for microseconds, not seconds

### Command Collection Strategy

For each device, the tool collects five commands in sequence:

| Command | Purpose | Fallback |
| :--- | :--- | :--- |
| `show version` | Extract hostname, OS version, uptime | Use management IP if hostname parse fails |
| `show interfaces` | Parse interface types, error counters, activity | Use TextFSM if available; use internal parser otherwise |
| `show interfaces status` | Extract port mode, VLAN, status using fixed-width parsing | Built-in fallback parser (no external dependency) |
| `show power inline` | Collect PoE admin/oper state and power draw | Empty dict if device is non-PoE or command fails |
| `show cdp/lldp neighbors detail` | Detect peer devices on each port | Boolean flag (true if neighbor present) |

**Why This Command Set?**
- Comprehensive but minimal: each command provides data no other command offers
- Covers the three dimensions of port health: *configuration* (mode/VLAN), *activity* (errors, last input), *attachment* (PoE, neighbors)

### The Intelligence Layer: Port Classification

```python
def classify_port(interface_record):
    """
    Assign a port to one of three categories:
    - 'access': Single VLAN, typically hosts
    - 'trunk': Multiple VLANs, typically uplinks
    - 'routed': No VLAN (layer 3), typically inter-device links
    """
```

**Classification Logic:**

From `show interfaces status`, inspect the VLAN column:

- If `trunk` or `rspan` → **Trunk**
- If `routed` → **Routed**
- Otherwise → **Access**

**Why This Matters:**
- Different port types require different stale-detection rules
- Access ports should be connected to hosts; trunk ports connect infrastructure
- This classification enables intelligent filtering and reporting

---

## 🛑 Stale Port Detection: Conservative & Accurate

### The Stale Logic Philosophy

A port is "stale" if it's not serving its intended purpose. But **false positives are dangerous**—flagging an active backup link as stale can trigger unnecessary troubleshooting.

The tool uses a **conservative, multi-condition approach**:

### For Access Ports (the Primary Target)

**Rule 1: Connected ports**
- Mark stale **ONLY** if `Last input ≥ <stale-days>`
- Example: A port with last activity 45 days ago (and stale threshold is 30 days) is flagged

**Rule 2: Disconnected ports**
- Mark stale **ONLY** if **BOTH** conditions hold:
  1. No PoE power draw (field is blank, `-`, or `0.0`)
  2. No LLDP/CDP neighbor present
- Example: A port that's admin-down, drawing no power, and has no neighbor → stale
- Counterexample: A port that's admin-down but still has a PoE device connected → NOT stale (device may be powered-off for maintenance)

### For Trunk & Routed Ports

These are typically infrastructure links and are **not subject to stale flagging**. False positives on trunks cause unnecessary escalations.

### Example Scenarios

| Status | Last Input | PoE Draw | Neighbor | Access Stale? | Trunk Stale? |
| :--- | :--- | :--- | :--- | :--- | :--- |
| connected | 45 days ago | 15W | Yes | ✅ YES | ❌ NO |
| notconnect | 60 days | - | No | ✅ YES | ❌ NO |
| notconnect | 60 days | 20W | No | ❌ NO | ❌ NO |
| admin down | 90 days | - | No | ✅ YES | ❌ NO |
| admin down | 1 day | 30W | No | ❌ NO | ❌ NO |
| trunk | - | - | Yes | ❌ (N/A) | ❌ NO |

---

## 📊 Data Collection: What Gets Parsed

The tool enriches each port with **11+ attributes** from four different sources:

### From `show version`
- Hostname (fallback: management IP)
- OS version
- Uptime
- Serial number

### From `show interfaces` (TextFSM or internal parser)
- Interface type (Gigabit, Fast Ethernet, etc.)
- Input errors, output errors, CRC errors
- Last input time (seconds ago, if available)
- MTU, duplex, speed

### From `show interfaces status` (fixed-width parser)
- Port status (connected, notconnect, err-disabled, admin down)
- Operational mode (access, trunk, routed)
- VLAN assignment
- Port description

### From `show power inline`
- PoE admin state (on, off, auto)
- PoE operational state (on, off, faulty)
- Power draw in watts
- Device class

### From `show cdp/lldp neighbors detail`
- Boolean: "Is a neighbor present on this port?"

---

## 📤 Excel Output: Strategic Intelligence

### The SUMMARY Sheet (First)

Provides an at-a-glance view of your access layer health:

| Column | Purpose | Example |
| :--- | :--- | :--- |
| Device | Switch hostname | `access-01.nyc` |
| Mgmt IP | Management interface | `10.1.1.51` |
| Total Ports (phy) | Physical port count | `48` |
| Access Ports | Count of access-mode ports | `42` |
| Trunk Ports | Count of trunk-mode ports | `4` |
| Routed Ports | Count of routed ports | `2` |
| Connected | Access ports with "connected" status | `38` |
| Not Connected | Access ports with "notconnect" status | `3` |
| Admin Down | Access ports admin-down | `1` |
| Err-Disabled | Access ports err-disabled | `0` |
| % Access of Total | Percentage of total that are access | `87.5%` |
| % Trunk of Total | Percentage of total that are trunk | `8.3%` |
| % Routed of Total | Percentage of total that are routed | `4.2%` |
| % Connected of Total | Percentage of all ports that are connected | `79.2%` |

**Final Row:** A TOTAL row sums numeric columns and calculates weighted percentages across all devices.

### Per-Device Sheets (One per Switch)

Detailed port-by-port inventory with filters and conditional formatting:

| Column | Purpose | Applied Formatting |
| :--- | :--- | :--- |
| `Interface` (normalized) | Gi1/0/1 (short form) | — |
| `Description` | Port label/purpose | — |
| `Status` | connected / notconnect / admin down / err-disabled | 🟢 Green if connected; 🔴 Red if disconnected/error; ⚪ Gray if admin-down |
| `Mode` | access / trunk / routed | — |
| `VLAN` | Access VLAN or trunk range | — |
| `Speed` | 100M, 1G, 10G, etc. | — |
| `Duplex` | Full / Half | — |
| `Input Errors` | Count of input errors | 🔴 Red if > 0 |
| `Output Errors` | Count of output errors | 🔴 Red if > 0 |
| `CRC Errors` | Count of CRC errors | 🔴 Red if > 0 |
| `Last Input` | Time since last activity | — |
| `PoE Power (W)` | Power draw in watts | 🟢 Green if > 0 |
| `PoE Oper` | on / off / faulty | — |
| `PoE Admin` | on / off / auto | — |
| `LLDP/CDP Neighbor` | Boolean | — |
| `Stale (≥N d)` | Boolean based on threshold | 🔴 Red if TRUE |

### Professional Formatting

Every sheet includes:

- **Frozen header** at row 2 (first row is device summary)
- **AutoFilter** on all columns (enable filtering with one click)
- **Auto-sized columns** with sensible min/max widths (readable without adjustment)
- **Conditional formatting** for quick visual scanning:
  - Status = connected → 🟢 green background
  - Status = notconnect / err-disabled → 🔴 red background
  - Status = admin down → ⚪ gray background
  - Any error counter > 0 → 🔴 red
  - PoE Power > 0 → 🟢 green
  - Stale = TRUE → 🔴 red

**Design Philosophy:** Operations teams can scan the workbook visually, spot red flags immediately, and drill into details only where needed.

---

## ⚙️ Performance & Concurrency

### ThreadPoolExecutor Pattern

The tool uses Python's `concurrent.futures.ThreadPoolExecutor` with a configurable worker count (default 10):

```python
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(audit_device, ip) for ip in device_list]
    for future in as_completed(futures):
        result = future.result()  # Block until job completes
```

**Why This Approach?**
- No complex thread management; the executor handles queuing and lifecycle
- Each device audit is independent; one failure doesn't stop others
- Progress bar updates in real-time as jobs finish
- Easy to tune: just change `max_workers`

### Tuning Guidance

| Scenario | Worker Count | Rationale |
| :--- | :--- | :--- |
| Low-latency LAN (< 1ms RTT) | 20+ | Fast round-trips; can handle many concurrent connections |
| Enterprise WAN (50+ ms RTT) | 5-10 | Each connection waits longer; don't overwhelm distant devices |
| Old/constrained devices (CPU, line rate) | 3-5 | Some devices have per-session limits; play it safe |
| Jump host with rate limits | 4-5 | Bastion hosts often throttle connections |

**Real-World Example:**
```bash
# Conservative: 50 switches with WAN + jump host
python -m main -w 4 --stale-days 90 -d devices.txt -o result.xlsx

# Aggressive: 20 switches on LAN, no bastion
python -m main -w 15 --direct -d devices.txt -o result.xlsx
```

---

## 🛰️ Jump Host (Bastion) Integration

### The Problem It Solves

Many enterprise networks use a "jump host" or "bastion" pattern: workstations cannot SSH directly to infrastructure; they must tunnel through a gateway.

**Example Network Topology:**

```
Your Workstation
    ↓
Jump Host (gateway.example.com) [SSH access allowed]
    ↓
Access Switches (10.x.x.x) [SSH only allowed from jump host]
```

### How It Works

The `JumpManager` component maintains a persistent SSH connection to the jump host, then proxies all device connections through it:

1. **Session 1:** SSH from workstation to `gateway.example.com` (using primary credentials)
2. **Session 2 (forwarded):** From gateway's perspective, SSH to `access-01` (using device credentials)
3. All traffic between gateway and switch is encrypted; the tunnel is transparent to your code

### Configuration

In `Modules/config.py`:

```python
JUMP_HOST = "gateway.example.com"  # Set to None to disable
JUMP_USER = "automation"            # Defaults to primary_user if not set
```

Command-line override:
```bash
# Use jump host from config
python -m main -d devices.txt -o audit.xlsx

# Skip jump host and connect directly
python -m main --direct -d devices.txt -o audit.xlsx
```

### Why Persistent Sessions Matter

Creating a new SSH tunnel for each of 50 devices would waste bandwidth and time:
- 50 tunnel creations × 2 seconds each = 100 seconds overhead
- Single persistent tunnel + 50 device sessions = ~5 seconds overhead

The `JumpManager` creates one tunnel and reuses it for all devices.

---

## 🗂️ Device List Format

The tool accepts a simple plain-text inventory file. Each line is one device:

```
# devices.txt - access switch inventory

# NYC site
10.10.10.11
access-01.nyc
access-02.nyc

# LA site (comments allowed)
192.168.50.10
access-03.la    # backup link

# Lines starting with # are ignored
# Blank lines are ignored
```

**Why This Format?**
- No JSON/YAML parsing overhead
- Can be auto-generated from Netbox, ServiceNow, or spreadsheets
- Easy to edit by hand
- Comments and blank lines are natural

---

## 🚀 Quick Start

### 1. Install Dependencies

```bash
pip install netmiko paramiko pandas openpyxl pywin32
```

On Windows, you may also need to run (for Credential Manager access):
```bash
pywin32_postinstall -install
```

### 2. Create Device List

```bash
# devices.txt
10.10.10.11
10.10.10.12
10.10.10.13
```

### 3. Configure Jump Host (Optional)

Edit `Modules/config.py`:
```python
JUMP_HOST = "gateway.example.com"
```

### 4. Run the Audit

```bash
# Basic: use jump host from config, default 10 workers
python -m main -d devices.txt -o access_audit.xlsx

# Skip jump host, use 5 workers, 60-day stale threshold
python -m main --direct -w 5 --stale-days 60 -d devices.txt -o results.xlsx

# Verbose debugging
python -m main --debug -d devices.txt -o audit.xlsx
```

---

## 🧭 CLI Reference

```
python -m main [OPTIONS]

OPTIONS:
  -d, --devices FILE       (required)  Path to device list file
  -o, --output FILE        (optional)  Output Excel filename (default: audit.xlsx)
  -w, --workers N          (optional)  Max concurrent device sessions (default: 10)
  --stale-days N           (optional)  Days threshold for stale access ports (default: 30, set 0 to disable)
  --direct                 (optional)  Skip jump host; connect directly
  --debug                  (optional)  Enable verbose logging
```

---

## 🧪 What the Script Collects (Per Device)

| Data | Source | Fallback Behavior |
| :--- | :--- | :--- |
| Hostname | `show version` \| include ^hostname | Use management IP |
| IOS Version | `show version` (parse major.minor) | Blank if parse fails |
| Uptime | `show version` | Blank if parse fails |
| Per-port mode | `show interfaces status` | Assume access if parse fails |
| Per-port VLAN | `show interfaces status` | Blank if not applicable |
| Per-port status | `show interfaces status` | Blank if parse fails |
| Interface type | `show interfaces` + TextFSM | Use internal parser if TextFSM unavailable |
| Error counters | `show interfaces` (InputErrors, OutputErrors, CRCErrors) | 0 if not found |
| Last input | `show interfaces` (activity timer) | Blank if not present |
| PoE power draw | `show power inline` | Blank if not PoE capable |
| Neighbor presence | `show cdp neighbors detail` or `show lldp neighbors detail` | False if command unavailable |

---

## 🐛 Logging, Debugging & Error Handling

### Debug Mode

Enable with `--debug` flag:

```bash
python -m main --debug -d devices.txt -o audit.xlsx
```

Prints:
- Jump host connection details
- Per-device SSH session info
- Command execution traces
- Parsing results
- Progress events

### Per-Device Error Isolation

If Device A fails, Device B continues:

```
[SUCCESS] 10.10.10.11 (access-01)
[FAIL]    10.10.10.12 (Authentication error)
[SUCCESS] 10.10.10.13 (access-03)
```

The Excel workbook includes:
- A row for the failed device in SUMMARY (with error text in place of metrics)
- A minimal sheet for the failed device (containing just the error message)

This ensures the workbook always represents all devices, even when some fail.

### Common Issues & Resolution

| Issue | Cause | Solution |
| :--- | :--- | :--- |
| `Authentication failed` | Wrong credentials in Credential Manager | Delete entry in Windows Credential Manager, re-run to re-prompt |
| `Connection refused` | SSH not reachable | Verify IP, check firewall, test reachability from jump host |
| `No such file or directory: devices.txt` | Wrong path | Use absolute path or verify file exists in current directory |
| `TextFSM templates not found` | `NET_TEXTFSM` environment variable not set | Set `NET_TEXTFSM` to point to NTC templates directory, or ignore (tool has fallback parser) |
| `Channel/SSH timeout` | Device too slow or network latency | Reduce `--workers` count |

---

## 🔧 Extending & Customizing

### Add More Commands

In `main.py`, locate the device audit function and add another command:

```python
lldp_output = netmiko_session.send_command('show lldp inventory')
# Parse and store in device_record['lldp_inventory'] = ...
```

### Change Conditional Formatting

In `main.py`, locate `_format_worksheet()` and modify fill colors, thresholds:

```python
# Example: flag ports with > 100 errors (not just > 0)
if interface['InputErrors'] > 100:
    ws[f'G{row}'].fill = PatternFill(fgColor='FF0000', fill_type='solid')
```

### Adapt Credentials for Linux/macOS

In `Modules/credentials.py`, replace the Windows Credential Manager logic with your preferred secure store:

```python
# Example: use Linux `pass` utility
def _read_linux_cred(target_name):
    result = subprocess.run(['pass', 'show', target_name], capture_output=True)
    if result.returncode == 0:
        return result.stdout.decode().strip()
    return None
```

### Tune Jump Host Settings

In `Modules/jump_manager.py`:

```python
ssh_client.get_transport().set_security_options(
    paramiko.util.log_to_file('/tmp/ssh_debug.log')  # Debug SSH handshake
)
```

---

## 🔒 Security Considerations

### Credential Storage

- ✅ **Recommended:** Windows Credential Manager, macOS Keychain, Linux pass/bitwarden
- ❌ **Never:** Plaintext config files, hardcoded in code, version-controlled files

### Network Access

- Run the tool from a secure workstation (not shared/untrusted systems)
- If using a jump host, ensure it's managed by your security team
- Generated Excel files contain sensitive infrastructure details; restrict read access

### Code Review

Before running on production devices:
1. Review `Modules/config.py` for your environment
2. Test on a small subset of devices first
3. Verify command set matches your device OS versions (some commands may not exist on older IOS)

---

## 🧩 Compatibility

| Aspect | Requirements |
| :--- | :--- |
| **Target Devices** | Cisco IOS/IOS-XE access and distribution switches |
| **Python Version** | 3.8+ |
| **SSH Protocol** | SSH v2 (standard on all modern Cisco devices) |
| **TextFSM** | Optional but recommended for robust parsing |
| **Windows Credential Manager** | Required on Windows; not available on Linux/macOS (tool falls back to prompts) |

### Device Type Compatibility

Tested and supported:
- Cisco Catalyst 2960-X
- Cisco Catalyst 3650
- Cisco Catalyst 9200 (IOS-XE)

Should work on:
- Any Cisco switch supporting the collected commands (`show version`, `show interfaces`, `show power inline`, `show cdp neighbors`)

---

## ✅ Real-World Examples

### Scenario 1: Quick Audit of a Single Site

```bash
# 10 switches, LAN, no jump host, default settings
python -m main -d site-a-switches.txt -o SiteA_Access_Audit.xlsx
```

**Result:** Excel workbook ready in 2-3 minutes with 10 device sheets + SUMMARY.

### Scenario 2: Enterprise WAN Audit with Jump Host

```bash
# 50 switches across 5 sites, WAN latency, going through jump host, conservative settings
python -m main -w 4 --stale-days 90 -d all-sites.txt -o enterprise_audit.xlsx
```

**Result:** All 50 sites audited in ~10 minutes (4 concurrent = less load on WAN and jump host).

### Scenario 3: Stale Port Investigation

```bash
# Focus on ports inactive > 60 days; include admin-down ports
python -m main --stale-days 60 -d sites.txt -o stale_analysis.xlsx
```

**Result:** Excel with red-highlighted stale ports; operations team immediately sees candidates for decommissioning.

### Scenario 4: Debug a Connectivity Issue

```bash
# Enable debug output to see SSH handshakes and command execution
python -m main --debug -d problem-devices.txt -o debug_output.xlsx 2>&1 | tee audit.log
```

**Result:** Detailed logs showing exactly where (and why) each device failed.

---

## 🧠 FAQs

### Q: Do I need TextFSM templates?
**A:** No, but they're recommended. The tool includes a robust fallback parser for `show interfaces status`. Without TextFSM, interface type and some error counters may be blank, but the core audit (port mode, VLAN, status, stale detection) still works perfectly.

### Q: How do credentials work on Linux/macOS?
**A:** The tool will prompt you interactively for credentials and store them in your system's keyring (Keychain on macOS, pass on Linux). You can also modify `Modules/credentials.py` to integrate with your password manager (Bitwarden, 1Password, Azure Key Vault, etc.).

### Q: How is "stale" determined for trunk ports?
**A:** Trunk ports are not subject to stale flagging. They're typically infrastructure links that should always be admin-up, even if they're not passing traffic at the moment.

### Q: What if a device's enable password is different from the login password?
**A:** The tool supports separate enable secrets via `get_enable_secret()` in `Modules/credentials.py`. Netmiko will attempt enable mode automatically if the initial connection is unprivileged.

### Q: Can I run this on a schedule (daily audits)?
**A:** Yes! The tool is designed for automation. You can schedule it with:
  - **Windows Task Scheduler:** Create a task that runs `python -m main ...` daily at 2 AM
  - **Linux cron:** `0 2 * * * cd /path && python -m main ...`
  - **Docker container:** Pre-load credentials into the container and run as a scheduled job

### Q: How do I handle devices with no management IP?
**A:** The tool requires SSH connectivity, which means devices must be addressable (either directly or via jump host). If a device has no management IP, it can't be audited. You can either:
  1. Add a management interface to the device
  2. Use DNS names (if resolvable from your workstation)
  3. Set up the jump host to have routing to that device's network

---

## 📝 License

GNU General Public License v3.0

## 👤 Author

Christopher Davies

---

> **Mission:** To empower network engineers with transparent, hardened Python tools that eliminate manual audits and expose infrastructure health at a glance.
