# Deep Dive: CDP Network Audit Tool
### "Cisco Python Automation, Explained Line-by-Line."

The **CDP Network Audit Tool** is a high-performance, multi-threaded discovery utility designed to map Cisco topologies with precision. It transforms raw Cisco Discovery Protocol (CDP) data into structured enterprise intelligence by combining Python automation with hardened security practices.

[:material-github: View Source Code on GitHub](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit){ .md-button .md-button--primary }

---

## 🎯 The Nautomation Prime Philosophy in Action

Before diving into the code, understand how every design decision reflects our three core principles:

### **Principle 1: Line-by-Line Transparency**
Every function in this tool is extensively documented. We don't just explain *what* the code does—we explain *why* it's structured this way and what engineering tradeoffs we made.

### **Principle 2: Hardened for Production**
You'll notice patterns like thread locks, exception handling, retry logic, and graceful cleanup. These aren't "nice to have"—they're essential for running automation on critical infrastructure without surprises.

### **Principle 3: Vendor-Neutral**
This tool is built on industry-standard libraries: **Netmiko** (SSH connection handling), **Paramiko** (SSH tunneling), **Pandas & OpenPyXL** (Excel reporting), and **TextFSM** (parsing). Your skills remain portable.

---

## 🏗️ Technical Architecture

The tool operates as a modular Python application with four primary components:

| Component | Responsibility | Why It Matters |
| :--- | :--- | :--- |
| **CredentialManager** | Secure credential collection and OS integration | Passwords stay out of code and config files |
| **NetworkDiscoverer** | Multi-threaded topology crawling via CDP | Discovers 50+ devices in seconds, not minutes |
| **ExcelReporter** | Professional, templated report generation | Maintains business branding and formatting |
| **Logging & Validation** | Pre-flight checks and operational visibility | Catches problems early; provides audit trail |

---

## 🔐 CredentialManager: Secure Credential Handling

### Why Credential Management Matters

**The Problem:** Network automation requires credentials. Storing them in plaintext files or hardcoding them in scripts is a security nightmare. Even prompting users every time is error-prone and doesn't scale to 10+ discovery jobs daily.

**The Solution:** Leverage native OS credential stores. Windows has Credential Manager, macOS has Keychain, Linux has pass. These are designed for exactly this use case and integrate with enterprise SSO/password managers.

**User Experience:** When you run CDP Network Audit for the first time, the script checks Windows Credential Manager for stored credentials. If none are found, it prompts you to enter your username and password. Once you provide them, the script saves them to Windows Credential Manager and uses them for the discovery process. On subsequent runs, the script retrieves the stored credentials automatically without prompting you again.

### `CredentialManager.__init__()`

```python
def __init__(self):
    # Read credential target names from environment (with sensible defaults)
    self.primary_target = os.getenv("CDP_PRIMARY_CRED_TARGET", "MyApp/ADM")
    self.answer_target = os.getenv("CDP_ANSWER_CRED_TARGET", "MyApp/Answer")
```

**Line-by-Line:**
- We read environment variables to allow customization per environment
- Default targets are `MyApp/ADM` (primary admin user) and `MyApp/Answer` (fallback user)
- This design means you can have different credential targets in dev vs. production

**Why This Matters:**
- Credentials are no longer part of the script or repository
- Environment variables can be injected at container runtime
- Easy to rotate target names without changing code

### `_read_win_cred(target_name: str)`

Reads encrypted credentials from Windows Credential Manager. Returns `(username, password)` tuple or `(None, None)` if not found.

**Key Points:**
- Only imports `win32cred` if available (Windows only)
- Handles both bytes and string returns for compatibility
- Decodes password from UTF-16LE (Windows internal format)
- Gracefully fails and returns None instead of crashing

**Why This Approach:**
- **No plaintext storage:** Credentials are encrypted by Windows
- **Cross-platform:** Non-Windows systems skip this and use prompts
- **Version-agnostic:** Works with multiple pywin32 versions

### `_write_win_cred(target: str, username: str, password: str) -> bool`

Writes credentials to Windows Credential Manager for future reuse. Users can optionally save credentials after first prompt.

**Key Points:**
- Password is encoded to UTF-16LE before storage (Windows requirement)
- `CRED_PERSIST_LOCAL_MACHINE` means credentials persist across sessions
- Failures are logged at DEBUG level (not alarming)
- Returns `True` on success, `False` on failure

**Why This Matters:**
- Users can avoid re-prompting on subsequent runs
- Credentials are encrypted and protected by Windows
- Optional save means users control persistence

### `get_secret_with_fallback(...) -> Tuple[str, str]`

The credential retrieval orchestrator with multi-step fallback:

1. **Try Credential Manager first** (if Windows)
2. **Fall back to interactive prompt**
3. **Optionally save to Credential Manager**

**Two-Credential Model:**
- **Primary:** Your main automation account (flexible username, likely AD-backed)
- **Answer:** A fallback user on each device (username is fixed as 'answer', local account)

**Why This Design:**
- Zero installation friction - first run prompts, subsequent runs use saved credentials
- Two credentials maximize success: primary fails → retry with answer
- Jump host always uses primary (tighter control)
- Device can fall back to answer user (weaker account)

### `prompt_for_inputs()`

Orchestrates all interactive input collection in one flow:

1. **Site name** - Used in Excel filename (max 50 chars)
2. **Seed devices** - Comma-separated IPs or hostnames
3. **Primary credentials** - Main automation account
4. **Answer credentials** - Fallback device account

---

## 🔌 NetworkDiscoverer: The Threaded Discovery Engine

### Why Parallel Discovery is Essential

**The Problem:** Discovering 50+ switches serially takes 10+ minutes. Each SSH connection is a round-trip: connect, execute, disconnect.

**The Solution:** Thread pool with 10 concurrent workers = 5x faster. 10 simultaneous connections instead of waiting for each one.

### Thread-Safe Data Accumulators

```python
# What we track (all thread-safe)
self.cdp_neighbour_details = []  # Parsed CDP entries
self.hostnames = set()           # Discovered hostnames for DNS
self.visited = set()             # IPs we've already audited
self.authentication_errors = set() # Auth failures
self.connection_errors = {}      # {IP: error_message}
self.dns_ip = {}                 # {hostname: resolved_ip}

# Protection mechanisms
self.visited_lock = threading.Lock()  # Protects queue management
self.data_lock = threading.Lock()     # Protects result accumulators
```

**Why Two Locks?**
- If we used one lock for everything, threads would block each other constantly
- Granular locks allow more independent work
- `visited_lock` for quick "is this already being processed?" checks
- `data_lock` for appending results

### `parse_outputs_and_enqueue_neighbors(...)`

This is the **intelligence** of the discovery engine. It decides which devices to audit next.

**Three-Step Process:**

**Step 1: Parse Device Context**
- Extract hostname, serial, uptime from `show version`
- Fall back to IP if parsing fails

**Step 2: Parse CDP Neighbors**
- Extract each neighbor's details (ports, capabilities, management IP)
- Store in thread-safe list

**Step 3: Apply Queueing Heuristic**

Only enqueue if ALL three conditions are true:

```python
if "Switch" in caps and "Host" not in caps and mgmt_ip:
```

**Why "Switch" in caps?**
- CDP capability strings like "Switch Router" identify infrastructure
- We only want to audit infrastructure nodes, not endpoints

**Why "Host" not in caps?**
- IP phones, printers, cameras also show up in CDP
- Their capability includes "Host" but we can't/shouldn't manage them

**Why mgmt_ip?**
- If a device doesn't advertise a management IP, we have no way to SSH to it
- Queueing it would just cause failures

**Example:**
```
Router (Switch, Router) + 10.1.1.5  → Queue it
IP Phone (Host) + 10.1.1.50          → Skip (endpoint)
Access Point (Host) + no Mgmt IP     → Skip (non-addressable)
```

### `_paramiko_jump_client(...) -> paramiko.SSHClient`

Creates a secure SSH connection to a jump/bastion host.

**Key Design Choices:**
- `WarningPolicy()` - Log warnings for unknown hosts (safer than AutoAddPolicy)
- Explicit password auth only - No SSH keys or agent (easier to audit)
- Consistent timeouts - All operations respect `CDP_TIMEOUT` setting
- Re-raise auth failures - Let caller handle credential issues

**Why WarningPolicy?**
- Accepts unknown hosts but logs warnings
- Catches potential man-in-the-middle attacks without crashing
- Production-ready security posture

### `_netmiko_via_jump(...)  -> ConnectHandler`

The **core connection function**. Handles both direct and jump-host connections.

**Credential Logic:**
```python
if primary:
    jump_user, jump_pass = primary_user, primary_pass
    device_user, device_pass = primary_user, primary_pass
else:
    jump_user, jump_pass = primary_user, primary_pass  # Jump always uses primary
    device_user, device_pass = answer_user, answer_pass  # Device uses fallback
```

**Why This Two-Credential Model:**
- Jump host always uses primary (tightest control)
- Device can use answer if primary fails
- Resilience: if your primary account is locked, answer account can still work

**Direct Connection:**
Simply pass device IP to Netmiko.

**Jump-Mediated Connection:**
1. Open Paramiko SSH to jump host
2. Create `direct-tcpip` channel (SSH tunnel) through jump to target
3. Wrap channel as socket
4. Pass socket to Netmiko for SSH auth

**Why direct-tcpip?**
- No need to open a listener on the jump host
- No port forwarding configuration required
- All traffic is inside the already-authenticated SSH session
- Secure and clean

### `run_device_commands(...) -> Tuple[str, str]`

Executes CDP and version commands on target device with fallback credentials.

**Strategy:**
1. Try with primary credentials
2. On auth failure, catch and retry with fallback (answer user on device)
3. Don't retry auth failures (credentials won't change between attempts)
4. Do retry transient errors (timeouts, SSH glitches) up to 3 times
5. Always disconnect in finally block (prevent socket leaks)

**Why This Approach:**
- Maximizes success rate with two-credential strategy
- Transient timeouts are retried (network glitches happen)
- Auth failures fail-fast (no point retrying)
- Finally block ensures resource cleanup

### `discover_worker(...) -> None`

The worker thread function. Multiple instances run concurrently.

**Loop:**
1. Get next host from queue (timeout=1.0 prevents hangs)
2. Recognize sentinel (None = shutdown signal)
3. Check if already visited (prevent duplicate work)
4. Execute discovery with up to 3 retries
5. Parse outputs and enqueue new neighbors
6. Always call `task_done()` or queue.join() will hang

**Why Sentinel Pattern?**
- None signals worker to exit gracefully
- Main thread sends one sentinel per worker
- Coordinated shutdown without races

**Why Check If Already Visited?**
- Concurrent workers might both process same IP
- Prevent duplicate discovery work
- Track with hostname and IP

**Why task_done() Is Critical:**
Without `task_done()`, `queue.join()` waits forever on main thread. This is a common source of hangs in multi-threaded code!

### `resolve_dns_parallel() -> None`

After discovery, resolve all discovered hostnames to IPs in parallel.

**Design:**
- ThreadPoolExecutor with 4-32 workers (based on CDP_LIMIT)
- Submit all resolutions concurrently
- Collect results as they complete (don't wait for slowest)
- Best-effort - failures are logged but don't block

**Why Separate from Discovery?**
- DNS lookups are independent
- Can run in smaller thread pool (4-32 vs. 10)
- Doesn't block discovery if DNS is slow

---

## 📊 ExcelReporter: Professional Reporting

### Why Professional Reporting Matters

**The Problem:** Raw CSV or unformatted Excel is not useful for business. Reports need context, formatting, branding.

**The Solution:** Use a pre-formatted Excel template. Write data into it while preserving all formatting, charts, filters, and branding.

### Template-Driven Approach

**Step 1: Copy Template**
```python
shutil.copy2(template, output_filename)
```

Preserve metadata (timestamps, permissions).

**Step 2: Stamp Metadata**
```python
ws["B4"] = site_name
ws["B5"] = date
ws["B6"] = time
ws["B7"] = seed1
ws["B8"] = seed2
```

Fill cells B4-B8 with audit metadata.

**Step 3: Append Data** 
```python
df.to_excel(writer, sheet_name="Audit", startrow=11, header=False)
```

Use `if_sheet_exists="overlay"` mode to append without destroying template.

Data starts at row 12 (after headers and metadata).

**Why This Approach:**
- **Template-driven:** Business controls formatting without touching code
- **Non-destructive:** Data is appended, template is preserved
- **Professional:** Charts, filters, styling all maintained
- **Automated:** No manual Excel editing required

### Multiple Output Sheets

| Sheet | Purpose | Rows |
| :--- | :--- | :--- |
| **Audit** | Main CDP data with local/remote ports, platforms | 12+ |
| **DNS Resolved** | Hostname → IP mappings | 5+ |
| **Authentication Errors** | List of IPs that failed auth | 5+ |
| **Connection Errors** | IPs with connection failures + error type | 5+ |

---

## 🔑 Key Design Patterns

### Pattern 1: Thread-Safe Data Accumulation
```python
with self.data_lock:
    self.results.append(new_data)
```
Only one thread updates shared data at a time.

### Pattern 2: Graceful Worker Shutdown
```python
for _ in range(num_workers):
    queue.put(None)  # Sentinel

# In worker
if item is None:
    return  # Exit gracefully
```

### Pattern 3: Retry with Fallback Credentials
```python
try:
    conn = connect(primary_user, primary_pass)
except AuthenticationException:
    conn = connect(answer_user, answer_pass)  # Fallback
```

### Pattern 4: Resource Cleanup in Finally
```python
try:
    conn = connect()
finally:
    if conn:
        conn.disconnect()  # ALWAYS happens
```

### Pattern 5: Template-Driven Reporting
Copy → stamp metadata → append data using overlay mode.

---

## 🎓 Learning Outcomes

After studying this code, you should understand:

✅ **Concurrent programming** — How thread pools and locks prevent race conditions  
✅ **SSH tunneling** — How direct-tcpip channels work and why they're safer  
✅ **Credential management** — OS-level credential stores vs. plaintext files  
✅ **TextFSM parsing** — How to extract structured data from CLI output  
✅ **Error handling** — Retry strategies and graceful degradation  
✅ **Excel automation** — Template-driven reporting with data overlay  
✅ **Network discovery** — CDP heuristics and neighbor crawling logic  

---

## 🚀 Distribution & Execution

Consistent with the **Nautomation Prime** delivery model, this tool is available in multiple formats:

* **Zero-Install Portable Bundle:** A self-contained package including the Python interpreter and all libraries (Netmiko, Pandas, TextFSM) for use on restricted Windows jump boxes.

* **Scheduled Docker Appliance:** A pre-built container designed for autonomous execution and periodic auditing.

---

## 📁 Repository & Downloads

Ready to audit your own network? Access the hardened source code and pre-configured templates below.

* **[:material-github: View Full Repository](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit)**: Access the code, TextFSM templates, and Excel master.
* **[:material-download: Download Latest Release](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit/archive/refs/heads/main.zip)**: Get a clean ZIP of the production-ready files.

---

## 📈 Performance Tuning

| Scenario | Configuration | Rationale |
| :--- | :--- | :--- |
| Fast LAN, many devices | `CDP_LIMIT=20`, `CDP_TIMEOUT=5` | High concurrency, short timeouts work |
| Slow WAN link | `CDP_LIMIT=5`, `CDP_TIMEOUT=30` | Fewer threads prevent overwhelming network; higher timeout for round-trip delay |
| Mixed (some LAN, some WAN) | `CDP_LIMIT=10`, `CDP_TIMEOUT=10` | Balanced defaults |
| Device with high CPU | `CDP_LIMIT=3-5` | Fewer threads prevent overwhelming device |

---

## 🎬 Next Steps

1. **Clone the repository:** `git clone https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit`
2. **Read the README** for installation and configuration
3. **Set up credentials** in Windows Credential Manager
4. **Run your first discovery** against a test device
5. **Review the Excel output** to understand the report format

Once comfortable, customize the discovery heuristics for your specific topology.

---

> **Mission Statement:** To empower engineers through Python-driven transparency and provide enterprises with hardened automation that eliminates error and accelerates growth.
