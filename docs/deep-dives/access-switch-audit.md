---
title: Access Switch Port Audit Tool - Deep Dive
description: Enterprise port intelligence tool with PoE monitoring, stale port detection, and professional Excel reporting. Modular Python automation for Cisco access switches.
tags:
  - Deep Dive
  - Access Switch
  - Port Audit
  - PoE
  - Excel
  - YAML
---

# Deep Dive: Access Switch Port Audit Tool
### "Enterprise Port Intelligence, Distilled to Pure Python."

A modular Python utility that connects to Cisco switches (optionally through an SSH jump host), collects comprehensive interface details, PoE information, and neighbor presence, then exports a professional, filters-only Excel workbook with a SUMMARY sheet and one sheet per device. Built for production reliability with **YAML-based configuration**, **intelligent fallback parsing**, and **customizable credential management**.

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
├── config.yaml             # YAML configuration file (NEW!)
├── README.md
└── Modules/
    ├── __init__.py
    ├── config_loader.py    # YAML config parser with type-safe properties
    ├── credentials.py      # Secure credential retrieval + fallbacks
    ├── jump_manager.py     # SSH jump host (bastion) support
    └── netmiko_utils.py    # Netmiko connection wrapper(s)
```

> **Note:** The `Modules/*.py` files encapsulate most environment-specific behaviour. The new `config_loader.py` provides a clean interface to YAML-based configuration with validation and type safety.
>
> **Key Improvement:** Configuration is now in human-readable YAML format instead of Python code, making it safer and more accessible to non-developers.

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

## ⚙️ Configuration System

The tool uses a **modern YAML-based configuration system** introduced in recent updates.

### YAML Configuration File (config.yaml)

All configurable settings are centralized in `config.yaml` at the project root. The configuration is loaded via `Modules/config_loader.py` which provides type-safe property accessors.

**Key Configuration Categories:**

**1. Network Settings:**
```yaml
network:
  jump_host: "jump-gateway.example.com"  # Default bastion/jump host
  device_type: "cisco_ios"                # Netmiko device type
  ssh_port: 22                            # SSH port
  read_timeout: 30                        # Command read timeout
```

**2. Credential Settings:**
```yaml
credentials:
  cred_target: "MyApp/ADM"  # Windows Credential Manager target
  enable_target: ""          # Optional enable secret target
```

**3. Performance & Concurrency:**
```yaml
concurrency:
  default_workers: 10        # Max concurrent device sessions
  retry_attempts: 3          # Connection retry count
  retry_base_wait: 2         # Base wait time for exponential backoff
```

**4. Excel Output:**
```yaml
output:
  default_filename: "audit.xlsx"  # Default output filename
  
excel_formatting:
  min_column_width: 10
  max_column_width: 50
```

**5. Stale Port Detection:**
```yaml
stale_detection:
  default_stale_days: 30  # Days threshold for stale ports
```

### Why YAML Configuration?

| Benefit | Explanation |
| :--- | :--- |
| **Human-Readable** | No Python knowledge required to modify settings |
| **Version Control Friendly** | Plain text format works seamlessly with Git |
| **Safer** | No code execution risk (pure data) |
| **Validated** | Config loader validates types and provides defaults |
| **Hierarchical** | Natural grouping of related settings |
| **Documented** | Inline comments explain each setting |

### Environment Variable Overrides

Specific settings can be overridden at runtime via environment variables (primarily for the jump host):

```powershell
# Override jump host at runtime
$env:JUMP_HOST = "temp-bastion.example.com"
```

> **Best Practice:** Use `config.yaml` for organizational defaults; use CLI arguments (`--direct`, `--workers`, etc.) for per-run overrides.

---

## 🚀 Quick Start: Using the Launcher (Recommended)

The repository now includes a **professional Windows batch launcher** (`run.bat`) that provides the easiest way to run the tool with default settings.

### Why Use the Launcher?

- **Zero configuration required** - Just double-click or run from command line
- **Automatic validation** - Checks for Python environment and required files before execution
- **Helpful diagnostics** - Clear error messages if something is missing
- **Professional interface** - Clean output with status indicators and progress messages
- **Safe execution** - Validates environment before running the script

### Using run.bat

**Option 1: Double-click**

Simply double-click `run.bat` in Windows Explorer to launch the tool with default behavior.

**Option 2: Command Line (Default Behavior)**

```cmd
run.bat
```

This runs the Access Switch Audit with all default settings from `config.yaml`.

### What the Launcher Does

1. **Validates the environment:**
   - Checks that the `portable_env` virtual environment exists
   - Verifies Python executable is present
   - Confirms `main.py` exists
   - Validates `config.yaml` and `devices.txt` are present

2. **Provides clear feedback:**
   - Shows [OK] for successful checks
   - Shows [WARNING] for missing optional files with option to continue
   - Shows [ERROR] for critical missing components
   - Displays helpful troubleshooting tips on failure

3. **Runs the tool:**
   - Activates the virtual environment
   - Executes the main script
   - Captures and displays the exit code
   - Provides common troubleshooting tips if errors occur

### Example Output

```
================================================================================
                  ACCESS SWITCH AUDIT TOOL
================================================================================

Starting validation checks...

[OK] Python Environment: Found at portable_env\Scripts\python.exe
[OK] Required support files found
[OK] All validation checks passed

================================================================================

Running Access Switch Audit...

================================================================================

[Script output appears here]

================================================================================

[SUCCESS] Script completed successfully

================================================================================
```

---

## 🚀 Advanced: Command Line with Arguments

For advanced users who need to **customize behavior beyond the defaults**, you can still run the tool directly with Python and command-line arguments.

### When to Use Command Line Arguments

Use `python main.py` with arguments when you need to:

- Override default settings from `config.yaml`
- Specify a different devices file
- Change output filename
- Adjust worker thread count
- Enable debug mode
- Force direct connections (bypass jump host)

### Method 1: Using the Launcher with Arguments

You can pass arguments to `run.bat` and they will be forwarded to the Python script:

```cmd
run.bat --devices my-switches.txt --output custom-audit.xlsx --workers 5
```

### Method 2: Direct Python Execution

Activate the virtual environment and run Python directly:

```bash
# Windows
portable_env\Scripts\activate
python main.py --devices my-switches.txt --output audit-report.xlsx

# Linux/macOS  
source portable_env/bin/activate
python main.py --devices my-switches.txt --output audit-report.xlsx
```

### Available Command-Line Arguments

| Argument | Description | Default |
|:---------|:------------|:--------|
| `--devices`, `-d` | Path to devices file | `devices.txt` |
| `--output`, `-o` | Output Excel filename | `audit.xlsx` |
| `--workers`, `-w` | Number of concurrent threads | 10 |
| `--stale-days` | Days threshold for stale ports | 30 |
| `--direct` | Skip jump host, connect directly | False |
| `--debug` | Enable debug-level logging | False |

**Example: Custom audit with direct connections:**

```bash
python main.py --devices critical-switches.txt --output critical-audit.xlsx --direct --debug
```

---

## 🏗️ Technical Architecture

The tool operates as a modular Python application with six primary components:

| Component | Responsibility | Why It Matters |
| :--- | :--- | :--- |
| **Config Loader** | YAML parsing and validation with type-safe properties | Settings are centralized, validated, and safe from code injection |
| **CredentialManager** | Secure credential retrieval from OS stores | Passwords never touch plaintext or config files |
| **JumpManager** | Persistent SSH tunnelling through a bastion host | Centralises network access control; supports air-gapped environments |
| **PortAuditor (Netmiko)** | Parallel device SSH connections and command collection | Audits 20+ switches in minutes, not hours |
| **PortIntelligence** | Multi-source port classification and risk flagging | Detects stale, misconfigured, or problematic ports automatically |
| **ExcelReporter** | Professional, templated workbook generation | Operations teams get insights immediately, not raw data dumps |

### Key Design Patterns

**1. Modular Configuration:**
- Settings separated from code (YAML vs Python)
- Config loader provides validation and type safety
- Environment-specific overrides supported

**2. Intelligent Fallback Parsing:**
- Primary: TextFSM templates (when available)
- Fallback: Custom fixed-width parsers
- Ensures reliability even without external dependencies

**3. Multi-Threaded Execution:**
- ThreadPoolExecutor with configurable worker count
- Thread-safe data accumulation with locks
- Per-device failure isolation

**4. Graceful Error Handling:**
- Exponential backoff retry logic
- Per-device error capture (doesn't stop entire audit)
- Comprehensive logging for troubleshooting

---

## 📊 Intelligent Parsing: The Heart of the Tool

### Why Intelligent Parsing Matters

**The Problem:** Cisco CLI output varies by device model, IOS version, and platform. `show interfaces status` might be formatted differently on a Catalyst 2960 vs a 9300. TextFSM templates might not exist for your specific platform.

**The Solution:** Multi-tier parsing strategy with intelligent fallbacks.

### Parsing Strategy: TextFSM + Custom Fallback

```python
def get_interfaces_via_show_interfaces(conn) -> List[Dict[str, Any]]:
    """
    Use TextFSM to parse 'show interfaces' for all ports.
    Falls back gracefully if templates unavailable.
    """
    try:
        output = conn.send_command("show interfaces", use_textfsm=True)
        if isinstance(output, list):
            return output
        return []
    except Exception:
        return []  # Graceful degradation
```

**Why This Approach:**
- TextFSM provides structured parsing when templates exist
- Returns empty list (not exception) if parsing fails
- Main logic continues with custom parsers

### Custom Fixed-Width Parser: `parse_show_interfaces_status()`

This is the **authoritative source** for port mode and VLAN classification.

```python
def parse_show_interfaces_status(output: str) -> List[Dict[str, str]]:
    """
    Robust fixed-width parser for 'show interfaces status'.
    Handles:
    - Multiple header formats
    - Variable column widths
    - Missing/malformed data
    """
```

**Step 1: Identify Header Row**

```python
def is_header(ln: str) -> bool:
    return ("Port" in ln and "Status" in ln and "Vlan" in ln and "Speed" in ln)
```

**Why:** Header detection must be flexible. Different IOS versions capitalize differently.

**Step 2: Extract Column Positions**

```python
def _find_columns(header_line: str) -> Dict[str, slice]:
    """
    Calculate exact character positions for each column.
    Returns slice objects for substring extraction.
    """
    tokens = ["Port", "Name", "Status", "Vlan", "Duplex", "Speed", "Type"]
    positions = {}
    for i, tok in enumerate(tokens):
        start = header_line.find(tok)
        end = header_line.find(tokens[i+1]) if i+1 < len(tokens) else len(header_line)
        positions[tok.lower()] = slice(start, end)
    return positions
```

**Why This Matters:**
- Fixed-width parsing is more reliable than regex for tabular CLI output
- Dynamically calculated positions adapt to slight formatting variations
- Slice objects provide clean substring extraction

**Step 3: Parse Data Rows**

```python
for line in lines:
    if line.startswith(("--", "Port")) or not line.strip():
        continue  # Skip separators and empty lines
    
    record = {}
    for key, col_slice in slices.items():
        record[key] = line[col_slice].strip()
    
    # Normalize status values
    status_raw = record.get('status', '').lower()
    if 'connect' in status_raw:
        record['status'] = 'connected'
    elif 'notconnect' in status_raw:
        record['status'] = 'notconnect'
    elif 'disabled' in status_raw:
        record['status'] = 'disabled'
    elif 'err' in status_raw:
        record['status'] = 'err-disabled'
```

**Why Status Normalization:**
- Different IOS versions use slight variations ("connected" vs "connect")
- Normalized values enable reliable conditional formatting in Excel
- Consistent categorization across device types

### Port Mode Classification

```python
# Determine mode from VLAN column
vlan_value = record.get('vlan', '').lower()

if vlan_value in ('trunk', 'rspan'):
    mode = 'trunk'
elif vlan_value == 'routed':
    mode = 'routed'
else:
    mode = 'access'  # Default assumption
```

**Why This Logic:**
- VLAN column is the most reliable indicator of port mode
- Trunk ports show "trunk" or "rspan" in VLAN field
- Routed ports show "routed"
- Everything else is access (may show VLAN number)

---

## 🔌 PoE Enrichment: Multi-Source Data Fusion

### The PoE Challenge

**Problem:** PoE data (`show power inline`) uses different interface naming than `show interfaces status`. Example:
- Status command: `Gi1/0/1`
- PoE command: `GigabitEthernet1/0/1`

**Solution:** Interface name aliasing and multi-key lookups.

### Interface Name Normalization

```python
def normalize_ifname(ifname: str) -> Tuple[str, str]:
    """
    Normalize interface names to canonical short and long forms.
    Returns: (short_form, long_form)
    Example: "Gi1/0/1" → ("Gi1/0/1", "GigabitEthernet1/0/1")
    """
    # Extract prefix and port number
    m = re.match(r"([A-Za-z]+)([0-9/\.]+.*)", ifname)
    if not m:
        return (ifname, ifname)
    
    prefix_raw = m.group(1)
    rest = m.group(2)
    
    # Map to short form
    short_prefix = _IF_MAP.get(prefix_raw.lower(), prefix_raw)
    
    # Generate long form
    long_prefix = {
        "Gi": "GigabitEthernet",
        "Fa": "FastEthernet",
        "Te": "TenGigabitEthernet",
        "Eth": "Ethernet",
    }.get(short_prefix, prefix_raw)
    
    return (f"{short_prefix}{rest}", f"{long_prefix}{rest}")
```

**Why This Matters:**
- Enables reliable cross-command matching
- Handles all common Cisco interface types
- Works across different IOS versions and platforms

### Alias-Based Lookup

```python
def all_aliases(ifname: str) -> List[str]:
    """
    Return all possible alias strings for an interface.
    Used for PoE data matching.
    """
    short, long = normalize_ifname(ifname)
    return [short, long, ifname]  # Try all variations

# During PoE enrichment:
for alias in all_aliases(port_name):
    if alias in poe_map:
        poe_data = poe_map[alias]
        break
```

**Why Multiple Aliases:**
- Different commands use different naming conventions
- Maximizes successful PoE data correlation
- Prevents data loss due to naming mismatches

---

## 🚨 Stale Port Detection: Conservative Risk Assessment

### The Business Problem

**Scenario:** You have 1,000 switch ports. How do you identify which ones are truly unused vs. temporarily disconnected vs. connected to equipment that's powered off?

**False Positives Are Expensive:**
- Marking an active port as "stale" disrupts operations
- Users lose network access
- Help desk tickets spike

**False Negatives Waste Resources:**
- Unused ports consume switch capacity
- Security risk (unauthorized devices can plug in)

### Conservative Detection Strategy

```python
def _categorize_port(row: Dict[str, Any], stale_days: int) -> str:
    """
    Classify port as: active, stale, or available.
    Uses conservative logic to minimize false positives.
    """
```

**Rule 1: Only Classify Access Ports**

```python
mode = row.get('Mode', '')
if mode != 'access':
    return 'active'  # Trunk and routed ports are infrastructure
```

**Why:** Trunk and routed ports connect switches to each other. They should never be flagged as stale.

**Rule 2: Connected Ports — Check Activity**

```python
status = row.get('Status', '')
if status == 'connected':
    last_input_secs = row.get('Last Input Seconds')
    if last_input_secs and last_input_secs >= (stale_days * 86400):
        return 'stale'
    return 'active'
```

**Why:**
- Port is physically connected
- But hasn't passed traffic in N days
- Likely a powered-off device or misconfigured endpoint

**Rule 3: Disconnected Ports — Check for Indicators**

```python
if status in ('notconnect', 'disabled', 'err-disabled'):
    # Conservative: require BOTH conditions to flag as stale
    has_poe = row.get('PoE Power (W)')
    has_neighbor = row.get('LLDP/CDP Neighbor')
    
    poe_w = None
    if has_poe:
        try:
            poe_w = float(str(has_poe).split()[0])
        except:
            pass
    
    # Stale only if: no PoE draw AND no neighbor
    if (poe_w is None or poe_w == 0.0) and not has_neighbor:
        return 'stale'
    
    return 'available'  # May be in use (PoE or neighbor present)
```

**Why This Conservative Approach:**

| Indicator | Interpretation |
| :--- | :--- |
| **PoE draw > 0** | Device is powered (IP phone, camera, AP) |
| **LLDP/CDP neighbor** | Device is network-aware (switch, phone, AP) |
| **Both absent** | Likely unused cable or dead device |

**Example Scenarios:**

| Status | PoE | Neighbor | Last Input | Classification | Reasoning |
| :--- | :--- | :--- | :--- | :--- | :--- |
| connected | 7.0W | Yes | 10 days | **active** | IP phone actively drawing power |
| connected | 0W | No | 45 days | **stale** | Connected but no traffic for 45+ days |
| notconnect | 0W | No | N/A | **stale** | Disconnected, no indicators of use |
| notconnect | 6.5W | No | N/A | **available** | PoE device present (might be powered off) |
| notconnect | 0W | Yes | N/A | **available** | Neighbor detected (might be rebooting) |

### Time Parsing: Handling Cisco Duration Formats

```python
def _parse_last_input_seconds(s: str) -> float | None:
    """
    Parse Cisco 'Last input' timer into seconds.
    Handles: "00:01:23", "1d2h30m", "never"
    """
    s = (s or "").strip().lower()
    if not s or s == "never":
        return None
    
    # Format 1: hh:mm:ss
    if re.match(r"^\d{1,2}:\d{2}:\d{2}$", s):
        hh, mm, ss = s.split(":")
        return int(hh) * 3600 + int(mm) * 60 + int(ss)
    
    # Format 2: Compact duration (1y2w3d4h5m6s)
    m = _TIME_RE.fullmatch(s.replace(" ", ""))
    if m:
        y = int(m.group("y") or 0)
        w = int(m.group("w") or 0)
        d = int(m.group("d") or 0)
        h = int(m.group("h").rstrip("h") or 0)
        m_val = int(m.group("m").rstrip("m") or 0)
        s_val = int(m.group("s").rstrip("s") or 0)
        
        days = y * 365 + w * 7 + d
        return days * 86400 + h * 3600 + m_val * 60 + s_val
```

**Why Multiple Format Support:**
- Different IOS versions use different time formats
- Ensures accurate stale detection across all platforms

---

## 🔐 Credentials & Security

The script retrieves device credentials using `Modules/credentials.py`:

- **Primary:** Windows Credential Manager (target name from `config.yaml`: default `MyApp/ADM`)
- **Fallback:** Interactive prompt for username and password (secure, not echoed)
- **Enable secret:** Retrieved by `get_enable_secret()` if configured in `config.yaml`, otherwise not required

> **Note:** If you are running on Linux/macOS, ensure `credentials.py` prompts for credentials or implements your preferred secure store. On Windows, `pywin32` enables Credential Manager access.
>
> **Important:** Never hard-code credentials in the repository. Use the secure store or environment prompts.
>
> **Configuration:** Credential Manager targets are set in `config.yaml` under the `credentials` section:
>
> ```yaml
> credentials:
>   cred_target: "MyApp/ADM"  # Primary credential target
>   enable_target: ""          # Optional enable secret target
> ```

---

## 🛰️ Jump Host (Bastion) Behaviour

- `main.py` reads `jump_host` from the `network` section of `config.yaml`
- **Default:** the script uses the jump host if `--direct` is not supplied
- `--direct` will skip the jump host entirely and attempt direct SSH connections

Example configuration in `config.yaml`:
```yaml
network:
  jump_host: "jump-gateway.example.com"  # or "" to disable by default
```

The `JumpManager` maintains a persistent SSH session to the bastion and proxies device connections through it.

**How JumpManager Works:**

```python
class JumpManager:
    def __init__(self, jump_host: str, username: str, password: str):
        self.jump_host = jump_host
        self.username = username
        self.password = password
        self.client = None  # Paramiko SSH client
    
    def connect(self) -> None:
        """Establish persistent SSH connection to bastion."""
        self.client = paramiko.SSHClient()
        self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        self.client.connect(
            self.jump_host,
            username=self.username,
            password=self.password,
            timeout=10
        )
    
    def open_channel(self, target_ip: str, target_port: int):
        """Open direct-tcpip channel through bastion."""
        return self.client.get_transport().open_channel(
            'direct-tcpip',
            (target_ip, target_port),
            ('localhost', 0)
        )
```

**Why direct-tcpip Channel:**
- No port forwarding needed on bastion
- All traffic stays within authenticated SSH session
- Cleaner than local port forwarding
- Works with restrictive bastion configurations

---

## �️ Device List File

Provide a plain-text file with one device per line. Lines that are blank or start with `#` are ignored.

```
# devices.txt
192.0.2.11
192.0.2.12  # inline comments are not parsed; this whole token must be a host/IP only
core-switch-01
access-sw-22
```

> **Note:** Hostnames must be resolvable from the machine (or via the jump host, depending on your SSH setup).

---

## 🚀 Quick Start

1. Install dependencies (see Requirements).
2. Create `devices.txt` with your targets (see Device list file).
3. (Optional) Configure `Modules/config.py` with your `JUMP_HOST`.
4. Run the audit:

```bash
# Using jump host from config.yaml
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

- **Credentials:** Adapt `Modules/credentials.py` to your environment (Linux keyring, Azure Key Vault, etc.)
- **Jump host:** Tune `Modules/jump_manager.py` (keep-alive, ciphers, auth methods) as needed
- **Connection behaviour:** Modify `Modules/netmiko_utils.py` for device types, timeouts, or SSH options
- **Configuration:** Edit `config.yaml` to set organizational defaults:
  - `network.jump_host`: Default bastion server
  - `concurrency.default_workers`: Concurrent device sessions
  - `stale_detection.default_stale_days`: Stale port threshold
  - `credentials.cred_target`: Credential Manager target name
  - `output.default_filename`: Default Excel output filename
- **Output columns:** Adjust record construction in `main.py` (search for `detailed.append({...})`)
- **Conditional formatting:** Tweak `_format_worksheet()` in `main.py`
- **Parsers:** Modify `parse_show_interfaces_status()` or `parse_show_power_inline()` for custom parsing logic

**Example config.yaml for Enterprise:**

```yaml
network:
  jump_host: "bastion.corp.example.com"
  read_timeout: 45  # Slower WAN links
  
credentials:
  cred_target: "NetworkAudit/Production"
  
concurrency:
  default_workers: 20  # Fast discovery
  retry_attempts: 5    # More retries for flaky network
  
stale_detection:
  default_stale_days: 90  # Longer threshold
  
output:
  default_filename: "port_audit_report.xlsx"
  
excel_formatting:
  min_column_width: 12
  max_column_width: 60
```

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

---

## 🎓 Learning Outcomes

After studying this code, you should understand:

✅ **YAML Configuration Management** — How to separate configuration from code using YAML with Python  
✅ **Fixed-Width Parsing** — Reliable CLI output parsing without external dependencies  
✅ **Multi-Source Data Fusion** — Correlating data across different commands using interface name aliasing  
✅ **Conservative Risk Assessment** — Stale port detection logic that minimizes false positives  
✅ **Thread-Safe Concurrency** — Parallel device audits with proper lock management  
✅ **Intelligent Fallback Strategy** — TextFSM + custom parsers for maximum compatibility  
✅ **SSH Tunneling** — Jump host integration with Paramiko direct-tcpip channels  
✅ **Excel Automation** — Professional workbook generation with conditional formatting  
✅ **Exponential Backoff** — Retry logic for transient network failures  
✅ **Credential Management** — Secure OS-level credential storage integration  

### Key Code Patterns Demonstrated

**Pattern 1: Graceful Degradation**
```python
try:
    data = parse_with_textfsm(output)  # Preferred method
except:
    data = parse_with_custom_logic(output)  # Fallback
```

**Pattern 2: Multi-Key Lookup**
```python
for alias in all_aliases(interface_name):
    if alias in poe_map:
        poe_data = poe_map[alias]
        break
```

**Pattern 3: Thread-Safe Accumulation**
```python
with lock:
    results.append(new_data)  # Atomic operation
```

**Pattern 4: Conservative Classification**
```python
if condition_A and condition_B:  # Both must be true
    mark_as_risky()
else:
    mark_as_safe()  # Default to safe
```

**Pattern 5: Type-Safe Configuration**
```python
@property
def default_workers(self) -> int:
    return self._get_nested("concurrency", "default_workers", default=10)
```

---

## 🚀 Distribution & Execution

Consistent with the **Nautomation Prime** delivery model, this tool is available in multiple formats:

* **Zero-Install Portable Bundle:** A self-contained package including the Python interpreter and all libraries (Netmiko, Pandas, OpenPyXL) for use on restricted Windows jump boxes.

* **Scheduled Docker Appliance:** A pre-built container designed for autonomous execution and periodic port auditing.

* **Source Code:** Full access to customize parsing logic, add vendor-specific commands, or integrate with your CMDB.

---

## 📋 Licence

GNU General Public Licence v3.0

## 👤 Author

Christopher Davies

---

> **Mission:** To empower network engineers with transparent, hardened Python tools that eliminate manual audits and expose infrastructure health at a glance.
