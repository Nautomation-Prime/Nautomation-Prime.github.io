# Deep Dive: Access Switch Port Audit Tool
### "Enterprise Port Intelligence, Distilled to Pure Python."

A modular Python utility that connects to Cisco switches (optionally through an SSH jump host), collects interface details, PoE information, and neighbor presence, then exports a clean, filters-only Excel workbook with a SUMMARY sheet and one sheet per device.

[:material-github: View Source Code on GitHub](https://github.com/Nautomation-Prime/Access_Switch_Audit){ .md-button .md-button--primary }

---

## ✨ What This Tool Does

- Audits access ports across multiple Cisco switches in parallel
- Normalises interface names (e.g., `GigabitEthernet1/0/1` → `Gi1/0/1`) for cross-command matching
- Enriches interfaces with:
  - PoE draw and state (`show power inline`)
  - LLDP/CDP neighbour presence (`show lldp neighbors detail`, `show cdp neighbors detail`)
- Classifies port mode & VLAN using `show interfaces status` (access / trunk / routed)
- Flags stale access ports using conservative rules
- Exports Excel with an at-a-glance SUMMARY and one sheet per device, with filters, frozen header, column auto-size, and conditional formatting
- Shows a progress bar whilst running concurrent device jobs

> **Design note:** The workbook intentionally uses filters only (no Excel tables), and places SUMMARY first.

---

## 🎯 The Nautomation Prime Philosophy in Action

Before diving into the code, understand how every design decision reflects our three core principles:

### **Principle 1: Line-by-Line Transparency**
Every function in this tool includes explicit documentation of *what it does* and *why it's structured this way*. You'll see comments explaining the engineering tradeoffs—why we parse with TextFSM *and* maintain a fallback parser, why we use conservative stale-detection logic, and why conditional formatting in Excel matters for operations teams.

### **Principle 2: Hardened for Production**
Access layer audits run on infrastructure that cannot afford downtime. You'll notice patterns like concurrent connection pooling, per-device failure isolation, graceful fallbacks when commands fail, and secure credential rotation. These aren't "nice to have"—they're mandatory for enterprise reliability.

### **Principle 3: Vendor-Neutral**
This tool is built on industry-standard Python libraries: **Netmiko** (multi-device SSH), **Paramiko** (jump host tunnelling), **Pandas & OpenPyXL** (Excel generation), and **TextFSM** (intelligent parsing). Your skills remain portable across vendors.

---

## 🧱 Project Layout

```
.
├── main.py                 # CLI entry point
├── devices.txt             # Example device list
├── README.md
└── Modules/
    ├── __init__.py
    ├── config.py           # Static configuration (e.g., JUMP_HOST)
    ├── credentials.py      # Secure credential retrieval + fallbacks
    ├── jump_manager.py     # SSH jump host (bastion) support
    └── netmiko_utils.py    # Netmiko connection wrapper(s)
```

> **Note:** The `Modules/*.py` files encapsulate most environment-specific behaviour (jump host, credential storage, and SSH connection settings).

---

## 📦 Requirements

- **Python:** 3.8+
- **Python packages:**
  - `netmiko`
  - `paramiko`
  - `pandas`
  - `openpyxl`
  - `pywin32` (Windows only; used for Windows Credential Manager integration)

Install with pip:
```bash
pip install netmiko paramiko pandas openpyxl pywin32
```

### Optional but Recommended

- **TextFSM templates** (NTC templates) for robust parsing of `show interfaces` when `use_textfsm=True`.
  - If templates are available and the `NET_TEXTFSM` environment variable points to them, parsing accuracy improves.
  - If not available, the script still works and falls back where needed (e.g., it has its own fixed-width parser for `show interfaces status`).

---

## 🏗️ Technical Architecture

The tool operates as a modular Python application with four primary components:

| Component | Responsibility | Why It Matters |
| :--- | :--- | :--- |
| **CredentialManager** | Secure credential retrieval from OS stores | Passwords never touch plaintext or config files |
| **JumpManager** | Persistent SSH tunnelling through a bastion host | Centralises network access control; supports air-gapped environments |
| **PortAuditor (Netmiko)** | Parallel device SSH connections and command collection | Audits 20+ switches in minutes, not hours |
| **PortIntelligence** | Multi-source port classification and risk flagging | Detects stale, misconfigured, or problematic ports automatically |
| **ExcelReporter** | Professional, templated workbook generation | Operations teams get insights immediately, not raw data dumps |

---

## 🔐 Credentials & Security

The script retrieves device credentials using `Modules/credentials.py`:

- **Primary:** Windows Credential Manager (target name expected by default: `MyApp/ADM`)
- **Fallback:** Interactive prompt for username and password (secure, not echoed)
- **Enable secret:** Retrieved by `get_enable_secret()` if configured, otherwise not required

> **Note:** If you are running on Linux/macOS, ensure `credentials.py` prompts for credentials or implements your preferred secure store. On Windows, `pywin32` enables Credential Manager access.
>
> **Important:** Never hard-code credentials in the repository. Use the secure store or environment prompts.

---

## 🛰️ Jump Host (Bastion) Behaviour

- `main.py` reads `JUMP_HOST` from `Modules/config.py`
- **Default:** the script uses the jump host if `--direct` is not supplied
- `--direct` will skip the jump host entirely and attempt direct SSH connections

Example configuration in `Modules/config.py`:
```python
JUMP_HOST = "jump-gateway.example.com"  # or None to disable by default
```

The `JumpManager` maintains a persistent SSH session to the bastion and proxies device connections through it.

---

## �️ Device List File

Provide a plain-text file with one device per line. Lines that are blank or start with `#` are ignored.

```
# devices.txt
10.10.10.11
10.10.10.12  # inline comments are not parsed; this whole token must be a host/IP only
core-switch-01
edge-sw-22
```

> **Note:** Hostnames must be resolvable from the machine (or via the jump host, depending on your SSH setup).

---

## 🚀 Quick Start

1. Install dependencies (see Requirements).
2. Create `devices.txt` with your targets (see Device list file).
3. (Optional) Configure `Modules/config.py` with your `JUMP_HOST`.
4. Run the audit:

```bash
# Using jump host from Modules/config.py
python -m main --devices devices.txt --output access_port_audit.xlsx

# Direct connections (no bastion), 5 workers, different stale threshold
python -m main --direct -w 5 --stale-days 60 -d devices.txt -o results.xlsx

# Verbose debugging
python -m main --debug -d devices.txt
```

---

## 🧭 CLI Reference

`main.py` exposes the following command-line options:

```
--devices, -d    (required)  Path to the devices file (one IP/hostname per line; '#' comments allowed)
--output,  -o    (optional)  Output Excel file name. Default: audit.xlsx
--workers, -w    (optional)  Max concurrent device sessions (threads). Default: 10
--stale-days     (optional)  Days threshold for stale access ports. 0 disables stale flagging. Default: 30
--direct         (optional)  Connect directly (do not use jump host)
--debug          (optional)  Enable verbose logging/prints
```

### Required vs Optional

- **Required:** `--devices`
- **Optional:** everything else

---

## �🔌 PortAuditor: The Threaded Collection Engine

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
| `show cdp/lldp neighbors detail` | Detect peer devices on each port | Boolean flag (true if neighbour present) |

**Why This Command Set?**
- Comprehensive but minimal: each command provides data no other command offers
- Covers the three dimensions of port health: *configuration* (mode/VLAN), *activity* (errors, last input), *attachment* (PoE, neighbours)

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

## 🛑 Stale Logic — How Ports Are Flagged

A conservative approach is used only for ports in `access` mode and when `--stale-days > 0`:

- **If Status = `connected`** → mark stale = True only if `Last input ≥ <stale-days>`
- **If Status ≠ `connected`** → mark stale = True when both conditions hold:
  1. No PoE draw (PoE power is blank/`-`/0.0), and
  2. No LLDP/CDP neighbour present on the port

This tends to avoid false positives on trunk/routed ports and on access ports actively in use.

> **Note:** You can disable stale flagging entirely by setting `--stale-days 0`.

---

## 🧪 What the Script Collects

For each device the script attempts to gather:

- **Hostname** (from `show running-config | include ^hostname` or CLI prompt fallback)
- **Interfaces** via TextFSM (`show interfaces`) when available
- **Port mode & VLAN** via a robust, fixed-width parser of `show interfaces status`
- **PoE details:** admin/oper state, power draw (W), class, device (`show power inline`)
- **Neighbour presence:** LLDP/CDP seen on the port (boolean)
- **Error counters:** input, output, CRC (from `show interfaces` parsed data)
- **Activity indicator:** "Last input" time (seconds parsed when present)

---

## 📤 Excel Output Structure

The workbook contains:

### 1) `SUMMARY` Sheet (First)

- One row per device, with a final TOTAL row (sums numeric columns)
- Columns include:
  - `Device`, `Mgmt IP`, `Total Ports (phy)`
  - `Access Ports`, `Trunk Ports`, `Routed Ports`
  - `Connected`, `Not Connected`, `Admin Down`, `Err-Disabled`
  - `% Access of Total`, `% Trunk of Total`, `% Routed of Total`, `% Connected of Total`

### 2) One Sheet Per Device

Columns typically include (when available):

- `Device`, `Mgmt IP`, `Interface` (long form), `Description`
- `Status` (normalised: connected / notconnect / administratively down / err-disabled)
- `AdminDown`, `Connected`, `ErrDisabled` (booleans for quick filters)
- `Mode` (access/trunk/routed), `VLAN`
- `Duplex`, `Speed`, `Type`
- `Input Errors`, `Output Errors`, `CRC Errors`
- `Last Input` (raw text)
- `PoE Power (W)`, `PoE Oper`, `PoE Admin`
- `LLDP/CDP Neighbour` (boolean)
- `Stale (≥<N> d)` (boolean)

### Formatting

- **Frozen header** (`A2`) and AutoFilter across all columns
- **Auto-sized columns** with sensible min/max widths
- **Conditional formatting:**
  - `Status = connected` → green
  - `Status = notconnect` / `err-disabled` → red
  - `Status = administratively down` → grey
  - Any `*Errors` > 0 → red
  - `PoE Power (W)` > 0 → green
  - `Stale (≥N d)` = TRUE → red

---

## ⚙️ Performance & Concurrency

- Uses a `ThreadPoolExecutor` with `--workers` threads (default 10)
- An event-driven progress bar updates as jobs start/finish
- Each device is independent; a failure on one does not stop others

The script prints an event-driven progress bar like:

```
Progress: [██████████░░░░░░░░░░░░] 12/30 started: 15/30
```

On completion, the Excel workbook is written to the filename you specify (default `audit.xlsx`).

---

## � Logging, Debug, and Errors

- Add `--debug` to surface additional prints (e.g., enable mode attempts, jump host info, file counts)
- Per-device errors are captured into the device's summary row (and a minimal sheet may be created with the error text so the workbook always reflects all devices)

**Common runtime issues & tips:**

- **Authentication failures** → check Credential Manager entry or typed credentials
- **SSH connectivity** → verify reachability from the workstation or via the jump host
- **TextFSM templates missing** → parsing still proceeds, but some fields may be blank
- **Channel/line rate limits on older devices** → consider lowering `--workers`

---

## 🔧 Extending and Customizing

- **Credentials:** adapt `Modules/credentials.py` to your environment (Linux keyring, Azure Key Vault, etc.)
- **Jump host:** tune `Modules/jump_manager.py` (keep-alive, ciphers, auth methods) as needed
- **Connection behaviour:** modify `Modules/netmiko_utils.py` for device types, timeouts, or SSH options
- **Output columns:** adjust record construction in `main.py` (search for `detailed.append({...})`)
- **Conditional formatting:** tweak `_format_worksheet()` in `main.py`

---

## 🔒 Security Considerations

- Prefer secure stores over plaintext
- Limit who can run the tool and who can read the generated Excel
- When using a jump host, ensure strong authentication and proper network segmentation

---

## 🧩 Compatibility

- **Target devices:** Cisco IOS/IOS-XE access and distribution switches reachable via SSH
- The tool relies on Netmiko; specify the right device type(s) inside `netmiko_utils.py`
- TextFSM/NTC templates significantly improve interface parsing fidelity but are not strictly required

### Tested Devices

This tool has been tested and verified on the following Cisco IOS and IOS-XE platforms:

- **Catalyst 9200 Series**
- **Catalyst 3650 Series**
- **Catalyst 3650C**
- **Catalyst 3650CG**
- **Catalyst 3650CX**
- **Catalyst 2960X Series**
- **Catalyst 2960 Series**

> **Note:** The tool should work with any Cisco IOS/IOS-XE device that supports the required show commands (interfaces, status, power inline, CDP/LLDP). The devices listed above have been explicitly tested and validated.

---

## ✅ Examples

```bash
# Basic, with jump host
python -m main -d devices.txt -o audit.xlsx

# Direct (no bastion), 20 workers, stale disabled
python -m main --direct -w 20 --stale-days 0 -d devices.txt -o audit.xlsx

# Conservative concurrency, higher stale threshold, verbose
python -m main -w 4 --stale-days 90 --debug -d devices.txt -o siteA.xlsx
```

---

## 🧠 FAQs

**Q: Do I need NTC TextFSM templates?**  
A: They are recommended for better `show interfaces` parsing. Without them, the script still works and uses its internal parser for `show interfaces status` and best-effort logic elsewhere.

**Q: Where do credentials come from?**  
A: On Windows, from Credential Manager (default target `MyApp/ADM`). Otherwise, you are prompted interactively or you can adapt `credentials.py` to your secret store.

**Q: How is `Mode` determined?**  
A: From `show interfaces status`: if VLAN column is `trunk`/`rspan` → `trunk`; if `routed` → `routed`; otherwise `access`.

**Q: How is a port considered stale?**  
A: Only for access ports and when `--stale-days > 0`. Connected ports are flagged stale only if `Last input ≥ N days`. Disconnected ports require both no PoE draw and no LLDP/CDP neighbour to be flagged stale.

---

## 📋 Licence

GNU General Public Licence v3.0

## 👤 Author

Christopher Davies

---

> **Mission:** To empower network engineers with transparent, hardened Python tools that eliminate manual audits and expose infrastructure health at a glance.
