# 🏗️ Architecture

Network design, virtual machine layout, and data flow of the SOC home lab.

---

## Design Principles

1. **Segmentation First** — Attacker, target, and monitoring networks are isolated.
2. **Realistic Detection** — The SIEM correlates events and escalates only meaningful signals.
3. **Automated Response** — Every high-severity alert is triaged and delivered automatically.

---

## Network Segmentation

| VMware Adapter | Network | Role | Subnet | pfSense Interface |
|:--------------:|:-------|:-----|:-------|:-----------------:|
| NAT | WAN | Internet uplink | 192.168.114.0/24 | EM0 |
| VMnet2 | LAN | Management & servers | 192.168.1.0/24 | EM1 |
| VMnet3 | SPAN | Traffic mirroring | — | EM2 |
| VMnet4 | Kali | Attacker network | 192.168.3.0/24 | EM3 |
| VMnet5 | Wazuh | SIEM segment | 192.168.4.0/24 | EM4 |
| VMnet6 | Splunk | Log collector | 192.168.5.0/24 | EM5 |

---

## Virtual Machines

| VM | OS | IP | Role |
|----|----|----|------|
| pfSense | FreeBSD | 192.168.1.254 | Router / Firewall |
| Wazuh Manager | CentOS-based | 192.168.1.150 | SIEM / XDR |
| n8n | Ubuntu 24.04 | 192.168.1.135 | SOAR Engine |
| Windows Server | Win Server 2019 | 192.168.1.100 | AD DC + Wazuh Agent |
| Kali | Kali Linux | 192.168.3.x (DHCP) | Attacker |

### Architecture Notes

- **Wazuh was migrated from VMnet5 to VMnet2 (LAN).** The SPAN network has no gateway and cannot initiate outbound API calls needed for the SOAR pipeline.
- **n8n runs in Docker** with a named volume for persistence.
- **The Windows Server** runs the Wazuh agent, forwarding Security Event ID 4625 (failed logon).

---

## Data Flow

Kali (192.168.3.x) launches Hydra RDP brute force against Windows Server.

Windows Event ID 4625 fires per failed logon attempt.

Wazuh agent forwards the event to the Manager on TCP/1514.

Manager correlates failures → alert severity ≥ level 10.

integratord invokes custom-n8n.py with the alert JSON path.

Script POSTs JSON to http://192.168.1.135:5678/webhook/wazuh-alerts

n8n receives the payload via its Webhook node.

IF node checks $json.body.rule.level ≥ 10 (TRUE → continue, FALSE → drop).

Set node extracts EmailSubject, Severity, AgentName, Description.

Gmail node sends HTML email via the Gmail API.

Analyst receives the alert in their inbox within seconds.


---

## Firewall Rules

| Rule | Interface | Proto | Source | Destination | Port | Action |
|------|-----------|:-----:|--------|-------------|:----:|:------:|
| Allow Wazuh → n8n | LAN | TCP | 192.168.1.150 | 192.168.1.135 | 5678 | Pass |
| Allow Agent → Manager | LAN | TCP | 192.168.1.100 | 192.168.1.150 | 1514, 1515 | Pass |

---

## Future Enhancements

- [ ] Slack / Discord / Teams notifications
- [ ] TheHive or Shuffle case management
- [ ] Zeek for network metadata capture
- [ ] Velociraptor for endpoint DFIR
- [ ] OpenCTI for threat intel enrichment
- [ ] LLM-based alert summarization in n8n
