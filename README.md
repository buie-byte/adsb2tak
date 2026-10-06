# ADS-B to TAK Server Guide

This project provides the software and step-by-step documentation to build a dedicated sensor node that decodes 1090 MHz ADS-B aircraft transmissions, enriches them with aircraft metadata, and forwards them as Cursor-on-Target (CoT) data to a TAK Server.

The system uses a Raspberry Pi 3B, an RTL-SDR Blog V3, and a Python script to create a real-time, enriched air picture.

---

## Installation Process

Please follow the steps in order. Each section is organized as a separate "tab" in this guide.

| **Tab** | **Description** |
|:---|:---|
| **[Step 1: Initial System Setup](./1-system-setup.md)** | Prepare the hardware and install the RTL-SDR software required to decode 1090 MHz aircraft signals. |
| **[Step 2: Network Configuration](./2-network-config.md)** | Configure the ZeroTier VPN and set up persistent host resolution to securely connect your node to the TAK Server. |
| **[Step 3: Application Setup](./3-application-setup.md)** | Establish the Python environment, install the aircraft metadata database, and configure mutual TLS certificates. |
| **[Step 4: Deployment & Validation](./4-deployment-validation.md)** | Perform end-to-end syntax and integration tests, then configure the gateway to run as a persistent system service. |
| **[Step 5: Operations Guide](./5-operations-guide.md)** | Learn how to monitor healthy live logs, troubleshoot common system faults, and apply software updates safely. |

---

## License & Disclaimer

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

**Disclaimer:** This repository is an independent installation guide and is not affiliated with, endorsed by, or connected to the Department of Defense, the TAK Product Center, or any official TAK development entity. Use these instructions at your own risk.
