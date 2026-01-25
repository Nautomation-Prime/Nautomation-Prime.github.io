# Deep Dive: CDP Network Audit Tool
### "Cisco Python Automation, Explained Line-by-Line."

The **CDP Network Audit Tool** is a high-performance, multi-threaded discovery utility designed to map Cisco topologies with precision. It transforms raw Cisco Discovery Protocol (CDP) data into structured enterprise intelligence by combining Python automation with hardened security practices.

### [Github page link to code](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit)

---

## 🏗️ Technical Architecture
The tool operates as a modular Python application built for reliability and safety in enterprise networks.

1. **Bootstrap & Asset Validation**: The script verifies that TextFSM templates and the Excel report master are present before initiating any SSH connections.
2. **Secure Authentication**: It utilizes a dual-tier credential model (Primary vs. Fallback) and integrates with the Windows Credential Manager.
3. **Threaded Discovery Engine**: A worker pool manages concurrent connections while protecting shared data via thread locks to prevent race conditions.
4. **Data Enrichment**: Post-discovery DNS resolution runs in a dedicated parallel pool to map hostnames to management IPs.
5. **Excel Reporting**: Results are written into a pre-formatted template using an overlay method that preserves existing branding and metadata.

---

## 🛡️ Hardened Security Logic

### 1. Jump-Host (Bastion) Support
To reach isolated subnets, the script establishes a secure Paramiko `direct-tcpip` channel.

```python
# From NetworkDiscoverer._netmiko_via_jump:
dest_addr = (target_ip, 22)
local_addr = ("127.0.0.1", 0)
channel = transport.open_channel("direct-tcpip", dest_addr, local_addr, timeout=self.timeout)
```

**Line-by-Line:** We create a tunnel inside an existing SSH session. This allows Netmiko to communicate with devices that have no direct external access.

**Safety:** This method avoids needing local listeners or exposing raw management ports on the jump box.

### 2. The Credential Manager Class
Security is enhanced by leveraging native OS credential stores rather than plaintext files.

```python
# From CredentialManager._read_win_cred:
cred = win32cred.CredRead(target_name, win32cred.CRED_TYPE_GENERIC)
user = cred.get("UserName")
blob = cred.get("CredentialBlob")
pwd = blob.decode("utf-16le") if blob else None
```

**Line-by-Line:** The script attempts to find stored credentials for a specific target name. If found, it decrypts the CredentialBlob directly from the Windows store.

**Transparency:** This provides a professional, "SSO-like" experience while ensuring passwords are never stored in the script or in plaintext on the disk.

## ⚙️ Discovery Engine: Explained Line-by-Line
### Thread-Safe Data Accumulation
Because up to 10 worker threads run simultaneously, the script must manage shared memory safely.

```python
# From NetworkDiscoverer.discover_worker:
with self.visited_lock:
    if host in self.visited:
        continue
    self.visited.add(host)
```

**Line-by-Line:** The visited_lock acts as a gatekeeper. Only one thread can check or update the visited list at a time.

**The Result:** This ensures the script never audits the same switch twice and never gets caught in an infinite loop between two neighboring switches.

### Neighbor Filtering Heuristics
The script "thinks" before it decides which device to audit next.

```python
# From NetworkDiscoverer.parse_outputs_and_enqueue_neighbors:
if "Switch" in caps and "Host" not in caps and mgmt_ip:
    if head not in self.visited_hostnames:
        self.visited_hostnames.add(head)
        self.host_queue.put(mgmt_ip)
```

**"Switch" in caps:** We target infrastructure nodes by checking CDP capability strings.

**"Host" not in caps:** We explicitly ignore endpoints like IP phones or printers that show up in CDP but cannot be managed via SSH.

**mgmt_ip:** If a device does not broadcast its management IP, it is a "dead end" for automation and is skipped to keep the queue clean.

## 📊 Automated Reporting Logic
The ExcelReporter class populates a professional report that respects your business formatting.

```python
# From ExcelReporter.save_to_excel:
ws1 = wb["Audit"]
ws1["B4"] = site_name
ws1["B5"] = date_now
ws1["B7"] = hosts[0] if hosts else ""
```

**Line-by-Line:** The script targets specific metadata cells (B4 to B8) in your master template to record Site Code, Date, and Seed Devices.

**Overlay Mode:** It uses the if_sheet_exists="overlay" mode to append data from row 12 onward, ensuring all original template headers and styles remain intact.

---

## 🚀 Distribution & Execution
Consistent with the **Nautomation Prime** delivery model, this tool is available in multiple formats:

* **Zero-Install Portable Bundle:** A self-contained package including the Python interpreter and all libraries (Netmiko, Pandas, TextFSM) for use on restricted Windows jump boxes.

* **Scheduled Docker Appliance:** A pre-built container designed for autonomous execution and periodic auditing.

---

> **Mission Statement:** To empower engineers through Python-driven transparency and provide enterprises with hardened automation that eliminates error and accelerates growth.