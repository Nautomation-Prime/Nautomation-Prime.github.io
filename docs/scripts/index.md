# Script Library
### Production-Ready Automation for Cisco Infrastructure

Welcome to the **Nautomation Prime Script Library**. Here you'll find open-source, hardened Python automation tools designed for enterprise Cisco deployments.

> **Important:** All scripts on this site are written in Python. You should have prior Python knowledge (variables, functions, loops, exception handling). Our scripts are designed for clarity and include comments, but we don't teach Python basics. If you're new to Python, learn the fundamentals first, then return here.

---

## 📚 Available Scripts

### CDP Network Audit Tool
**Status:** ✅ Available  
**Description:** A high-performance, multi-threaded discovery utility that maps Cisco topologies via CDP.

**Features:**
- Multi-threaded concurrent device discovery
- Jump-host (bastion) support for isolated networks
- Windows Credential Manager integration
- TextFSM template-based parsing
- Excel reporting with custom formatting

[📖 View Deep Dive Documentation](../deep-dives/cdp-audit.md) | [:material-github: GitHub Repository](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit)

---

## 🔄 Coming Soon

### IOS-XE Software Upgrade Orchestrator
Automated, intelligent firmware management for switch stacks with pre-flight verification and binary integrity checks.

### Access Switch Audit Suite
Comprehensive auditing tool for access layer switches with detailed Excel reporting and compliance mapping.

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
- **macOS:** Credentials are stored in Keychain
- **Linux:** Credentials are stored in `pass` or similar managers

Credentials are never stored in plaintext files or hardcoded in scripts.

See each repository's README for platform-specific instructions.

### Configuration

All scripts follow the **Nautomation Prime** philosophy of transparency and security:
- Credentials are stored in OS credential managers (Windows Credential Manager, etc.)
- Configuration files are well-documented with inline comments
- Pre-flight validation checks prevent unsafe deployments

### Support & Questions

For issues, feature requests, or questions about any script:
- Check the **Deep Dives** documentation for detailed explanations
- Open an issue on the respective GitHub repository
- [Contact Nautomation Prime](https://forms.office.com/Pages/ResponsePage.aspx?id=DQSIkWdsW0yxEjajBLZtrQAAAAAAAAAAAAO__ZCPnztUMEtPWTVHN0JQTjZMME5YTTgxMEhRN0MwQS4u) for consulting services

---

## 📖 The "Prime" Philosophy

All scripts in this library adhere to three core principles:

1. **Line-by-Line Transparency** - Every function is documented, every decision explained
2. **Hardened for Production** - Robust error handling, security best practices, pre-flight checks
3. **Vendor-Neutral** - Built on industry-standard libraries like Netmiko, Nornir, and TextFSM

> **Mission:** To empower engineers through Python-driven transparency and provide enterprises with hardened automation that eliminates error and accelerates growth.
