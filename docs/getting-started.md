---
title: Getting Started
description: Learn what Nautomation Prime offers and find the right path for your network automation journey. Quick start guide for deep dives, scripts, and services.
tags:
  - Getting Started
  - Guide
  - Tutorial
  - Onboarding
---

Welcome! This guide will help you understand what Nautomation Prime offers and how to get started.

---

## What is Nautomation Prime?

**Nautomation Prime** bridges the gap between complex Cisco infrastructure and streamlined Python-driven automation. We provide:

- **Production-ready automation scripts** explained line-by-line
- **Deep-dive technical guides** that teach you the "why" behind the code
- **Enterprise solutions** including Docker containers and portable bundles
- **Bespoke services** for custom automation needs

**Important:** This site is **not a Python tutorial.** We assume you already know Python basics (variables, functions, loops, exceptions, file I/O). Our goal is to teach you how to apply Python to network automation and provide a foundation you can transfer into your own scripts or learning journey.

---

## 🚀 Quick Start Paths

### I want to learn network automation

Start with our **[Technical Deep Dives](deep-dives/index.md)**. We provide comprehensive guides that explain Python automation concepts alongside real Cisco use cases.

**Recommended Reading Order:**

1. [CDP Network Audit Deep Dive](deep-dives/cdp-audit.md) - Learn about threading, security, and production-grade design

### I want to use pre-built scripts

Check out our **[Script Library](scripts/index.md)**. Each script comes with documentation and GitHub repositories for easy deployment.

**Popular Scripts:**

- [CDP Network Audit Tool](scripts/index.md) - Discover your Cisco topology with line-by-line transparency

### I need custom automation for my environment

Explore our **[Services](services.md)** page. We offer:

- Custom Python scripting tailored to your topology
- Portable bundles for restricted environments
- Docker containers for continuous automation
- ISE and Zero Trust automation

---

## 📋 Prerequisites

**Python Knowledge (Important!):**

- **This site assumes you already know Python.** We teach you how to apply Python to network automation, not how to learn Python itself.
- If you're new to Python, we recommend completing a Python fundamentals course first (Codecademy, Real Python, W3Schools, or similar).
- You should understand: variables, functions, loops, conditionals, exceptions, and basic file I/O.
- Our code is written for clarity (not brevity), so intermediate Python developers will follow along easily.

**Technical Requirements:**

- **Python 3.8+** (or use our [portable bundles](services.md#zero-install-deployment-portable-bundles) if Python isn't available)
- **Network access** to your Cisco devices
- **Credentials** for device authentication
- **SSH enabled** on target Cisco devices

---

## 🔐 Security & Credentials

Nautomation Prime follows enterprise security best practices:

✅ **Credentials stored in OS credential managers** (not plaintext files)  
✅ **No hardcoded secrets** in scripts  
✅ **Secure jump-host support** for isolated networks  
✅ **Full source code transparency** (no compiled binaries)

---

## 📚 Core Concepts

### The "Prime" Philosophy

Every tool, script, and guide adheres to three principles:

1. **Line-by-Line Transparency**
   - We explain the *why* behind the code, not just the *what*
   - Every design decision is documented
   - You'll understand your automation, not just run it

2. **Hardened for Production**
   - Robust error handling
   - Pre-flight safety checks
   - Enterprise-grade credential management
   - Thread-safe concurrent operations

3. **Vendor-Neutral**
   - Built on industry-standard libraries (Netmiko, Nornir, PyATS)
   - Your skills remain portable
   - Scripts can scale beyond Cisco if needed

---

## 🛠️ Common Tasks

### Deploy the CDP Network Audit Tool

1. Visit the [CDP Audit GitHub repository](https://github.com/Nautomation-Prime/Cisco_CDP_Network_Audit)
2. Read the [Deep Dive guide](deep-dives/cdp-audit.md) for understanding the architecture
3. Follow the README for installation and configuration
4. Run your first discovery against a test device

### Request Custom Automation

1. Document your use case and network topology
2. Contact us via [email](mailto:nautomationprime.f3wfe@simplelogin.com) or [LinkedIn](https://www.linkedin.com/company/nautomationprime)
3. Describe any constraints (e.g., restricted environments, specific platforms)
4. Receive a detailed proposal and timeline

### Use Portable Bundles (No Python Installation)

1. Request a custom bundle through our [services page](services.md#zero-install-deployment-portable-bundles)
2. Download the bundle to your workstation or USB drive
3. Extract and run directly—no installation needed
4. Full source code is included for auditing

---

## ❓ Frequently Asked Questions

**Q: Do I need Python installed to use Nautomation Prime tools?**  
A: Not necessarily! We offer portable bundles that run without Python installation. Ideal for restricted enterprise environments where Python may not be permitted.

**Q: Can you automate my specific network topology?**  
A: Absolutely! Our bespoke services cover custom scripting for any topology. Contact us via [email](mailto:nautomationprime.f3wfe@simplelogin.com) or [LinkedIn](https://www.linkedin.com/company/nautomationprime) to discuss your specific requirements.

**Q: Are these tools vendor-locked to Cisco?**  
A: Our tools are built on vendor-neutral libraries like **Netmiko** and **Nornir**. While designed for Cisco, the patterns and concepts apply across other vendors (Juniper, Arista, etc.).

**Q: How do I secure my credentials?**  
A: We leverage native OS credential managers (Windows Credential Manager, Keychain on macOS, pass on Linux). Passwords are never stored in plaintext files or hardcoded in scripts. When you run a script like CDP Network Audit for the first time, it will prompt you to save your credentials to Windows Credential Manager—just enter your username and password, and the script will store them securely. Future runs will use the stored credentials automatically.

**Q: What about support and updates?**  
A: All tools are maintained on GitHub with active development. Issues and feature requests can be filed directly on repositories. For enterprise support, [contact us](services.md).

---

## 📖 Next Steps

- **Learn:** Dive into a [Deep Dive](deep-dives/index.md)
- **Deploy:** Browse the [Script Library](scripts/index.md)
- **Build:** Explore [Services](services.md) for custom solutions
- **Connect:** Contact us via [email](mailto:nautomationprime.f3wfe@simplelogin.com) or [LinkedIn](https://www.linkedin.com/company/nautomationprime)

---

> **Our Mission:** To empower engineers through Python-driven transparency and provide enterprises with hardened automation that eliminates error and accelerates growth.
