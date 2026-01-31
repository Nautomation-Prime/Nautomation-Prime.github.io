# Script Library
### Production-Ready Automation for Cisco Infrastructure

Welcome to the **Nautomation Prime Script Library**. Here you'll find open-source, hardened Python automation tools designed for enterprise Cisco deployments.

> **Important:** All scripts on this site are written in Python. You should have prior Python knowledge (variables, functions, loops, exception handling). Our scripts are designed for clarity and include comments, but we don't teach Python basics. If you're new to Python, learn the fundamentals first, then return here.

---

## 📚 Available Scripts

### CDP Network Audit Tool
**Status:** ✅ Available  
**Description:** A threaded discovery utility that starts from seed Cisco devices and crawls the network using Cisco Discovery Protocol (CDP), producing structured Excel reports with professional formatting.

**Features:**
- Parallel discovery with configurable worker pool (via config.py or environment variable overrides)
- Centralised configuration with comprehensive config.py (200+ documented settings)
- Two-tier authentication (primary user with customisable fallback username)
- Jump server / bastion support (Paramiko channel + Netmiko sock)
- DNS enrichment for discovered hostnames
- Excel reporting from pre-formatted templates with multiple sheets
- Hybrid logging with optional logging.conf
- Up to 3 automatic retries for transient connectivity issues
- Comprehensive error tracking (authentication failures, connection errors)
- Extensive customisation options (credentials, paths, Excel formatting, DNS, logging, and more)

[📖 View Deep Dive Documentation](../deep-dives/cdp-audit.md) | [:material-github: GitHub Repository](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit)

---

### Access Switch Port Audit Tool
**Status:** ✅ Available  
**Description:** A production-hardened collector designed to map interface health and utilisation across your access layer.

**Features:**
- Parallel device SSH connections for high-speed audits
- Conservative "Stale Port" detection logic using PoE, neighbours, and input timers
- Multi-source port classification (Access vs. Trunk vs. Routed)
- Professional Excel workbooks with automated conditional formatting
- Full Jump-Host (Bastion) integration for restricted environments

[📖 View Deep Dive Documentation](../deep-dives/access-switch-audit.md) | [:material-github: GitHub Repository](https://github.com/Nautomation-Prime/Access_Switch_Audit)

---

## 🔄 Coming Soon

### Zero Touch Provisioning (ZTP) Tool
**Status:** 🚧 In Development  
**Description:** Automated deployment solution for Cisco devices that streamlines initial configuration and reduces deployment time from hours to minutes.

**Planned Features:**
- Automated device configuration from templates
- DHCP option integration for network-based provisioning
- Email notifications for deployment status and errors
- HTTP server integration for configuration and log file management
- Pre-flight validation and rollback capabilities
- Multi-device orchestration with dependency management
- Comprehensive logging with remote log collection

**Current Status:** Core functionality tested and validated. Additional features (email notifications, HTTP log server integration) under active development.

---

### IOS-XE Software Upgrade Orchestrator
**Status:** 🚧 In Development  
**Description:** Automated, intelligent firmware management for Cisco IOS-XE switch stacks that eliminates manual upgrade errors and reduces downtime through comprehensive pre-flight validation.

**Planned Features:**
- Pre-flight validation (disk space, compatibility, current version checks)
- Binary integrity verification (MD5/SHA checksums)
- Automated file transfer to target devices (SCP/TFTP/HTTP)
- Stack-aware upgrade orchestration with rolling restarts
- Version compliance reporting across the estate
- Rollback capability for failed upgrades
- Parallel upgrade support for multiple stacks
- Email notifications and comprehensive logging
- Integration with maintenance windows and change control systems

**Current Status:** Architecture and design phase. Feature set being finalized based on enterprise deployment requirements.

---

## 🛠️ Getting Started with Scripts

### Prerequisites
- Python 3.8+
- Netmiko or equivalent SSH library
- Network access to target devices
- Appropriate credentials/permissions

### Installation & Setup

Each script repository includes detailed installation instructions in its README. Typical workflow:

```bash
# Clone the repository
git clone https://github.com/Nautomation-Prime/<script-name>
cd <script-name>

# Install dependencies
pip install -r requirements.txt

# Run with --help to see options
python main.py --help
```

### Credential Management

Scripts use your operating system's native credential manager for secure authentication:

- **Windows:** CDP Network Audit prompts you to save credentials to Windows Credential Manager on first run. Enter your username and password when prompted, and the script will store them securely. Future runs use the stored credentials automatically.  
- **macOS:** Credentials are stored in Keychain - Upcoming  
- **Linux:** Credentials are stored in `pass` or similar managers - Upcoming  

Credentials are never stored in plaintext files or hardcoded in scripts.

See each repository's README for platform-specific instructions.

### Configuration

All scripts follow the **Nautomation Prime** philosophy of transparency and security:  
- Credentials are stored in OS credential managers (Windows Credential Manager, etc.)  
- Configuration files are well-documented with inline comments.  
- Pre-flight validation checks prevent unsafe deployments.  

### Support & Questions

For issues, feature requests, or questions about any script:  
- Check the **Deep Dives** documentation for detailed explanations.  
- Open an issue on the respective GitHub repository.  
- [Contact Nautomation Prime](https://forms.office.com/Pages/ResponsePage.aspx?id=DQSIkWdsW0yxEjajBLZtrQAAAAAAAAAAAAO__ZCPnztUMEtPWTVHN0JQTjZMME5YTTgxMEhRN0MwQS4u) for consulting services.  

---

## 📖 The "Prime" Philosophy

All scripts in this library adhere to three core principles:

1. **Line-by-Line Transparency** - Every function is documented, every decision explained  
2. **Hardened for Production** - Robust error handling, security best practices, pre-flight checks  
3. **Vendor-Neutral** - Built on industry-standard libraries like Netmiko, Nornir, and TextFSM  

> **Mission:** To empower engineers through Python-driven transparency and provide enterprises with hardened automation that eliminates error and accelerates growth.
