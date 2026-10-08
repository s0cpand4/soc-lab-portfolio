# 🛡️ SOC Home Lab — Wazuh → n8n → Gmail SOAR Pipeline

> A fully self-hosted Security Operations Center lab that detects real attacks with **Wazuh SIEM** and responds automatically via an **n8n SOAR pipeline** that delivers formatted Gmail alerts — built entirely from open-source tools at **$0 software cost**.

[![Wazuh](https://img.shields.io/badge/Wazuh-4.14-3fb950?style=flat-square)](https://wazuh.com)
[![n8n](https://img.shields.io/badge/n8n-Docker-58a6ff?style=flat-square)](https://n8n.io)
[![pfSense](https://img.shields.io/badge/pfSense-CE_2.6-d29922?style=flat-square)](https://pfsense.org)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docker.com)

🌐 **[View the full portfolio →](https://socpanda4.github.io/soc-lab-portfolio/)**

---

## 📖 Overview

Modern SOC teams don't just detect — they **respond**. This lab demonstrates a complete detection and automated response pipeline, from network-layer attacks launched from Kali Linux through to a formatted email landing in an analyst's inbox in under five seconds.

Built on **VMware Workstation Pro** with **pfSense** providing firewall, routing, and DHCP across six isolated network segments.

---

## 🏗️ Pipeline

┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ KALI │─▶│ WINDOWS │─▶│ WAZUH │─▶│ N8N │─▶│ GMAIL │
│ attacker │ │ target │ │ SIEM │ │ SOAR │ │ inbox │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘


---

## 🧰 Technology Stack

| Layer | Technology |
|-------|-----------|
| SIEM/XDR | Wazuh 4.14 |
| SOAR | n8n (Docker) |
| Firewall | pfSense CE 2.6 |
| Endpoint | Windows Server 2019 + Wazuh Agent |
| Attacker | Kali Linux (Hydra, Nmap) |
| Runtime | Ubuntu Server 24.04 + Docker |
| Auth | OAuth2 / Gmail API |
| Hypervisor | VMware Workstation Pro |

---

## 📂 Repository Structure
soc-lab-portfolio/
├── index.html # Portfolio page (GitHub Pages)
├── README.md # This file
├── LICENSE # MIT
├── docs/
│ ├── architecture.md # Network design and VM layout
│ ├── wazuh-setup.md # SIEM install + integration script
│ └── n8n-setup.md # Docker + OAuth2 walkthrough
├── integrations/
│ └── custom-n8n.py # Wazuh → n8n webhook forwarder
├── configs/
│ ├── docker-compose.yml # n8n container definition
│ └── ossec-integration.xml # ossec.conf integration snippet
└── screenshots/ # Dashboard, alerts, email proof


---

## 📚 Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | Network topology, VM layout, data flow |
| [Wazuh Setup](docs/wazuh-setup.md) | SIEM deployment and integration script |
| [n8n Setup](docs/n8n-setup.md) | Docker deployment and Gmail OAuth2 |
| [Portfolio Site](https://socpanda4.github.io/soc-lab-portfolio/) | Interactive project showcase |

---

## ✅ Validation Results

| Test | Result |
|------|:------:|
| Webhook delivery | ✅ 200 OK |
| Severity filtering (IF node) | ✅ Correct branch |
| Data transformation (Set node) | ✅ Fields populated |
| Gmail API auth (OAuth2) | ✅ Token issued |
| End-to-end alert (Wazuh) | ✅ Email delivered |
| Brute-force correlation (Hydra) | ✅ Rule fired |

---

## ⚠️ Disclaimer

This lab is built for **educational purposes only**. All attacks are simulated against VMs on an isolated private network. Never run these tools against systems you do not own or have explicit written authorization to test.

---

## 📜 License

MIT License — see [LICENSE](LICENSE) for details.

---
