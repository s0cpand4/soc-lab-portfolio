# 🛡️ Wazuh Setup & Integration

Deploy the Wazuh Manager and configure the custom Python integration that forwards high-severity alerts to n8n.

---

## Installation (OVA Method)

1. Import the Wazuh OVA into VMware Workstation Pro
2. Boot and log in with the default credentials:
   - **Username:** `wazuh-user`
   - **Password:** `wazuh`
3. Update the system:
   ```bash
   sudo dnf update -y

Installation (Package Method)

# Add the Wazuh repository
curl -sO https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo rpm --import GPG-KEY-WAZUH

# Add the repo entry
sudo tee /etc/yum.repos.d/wazuh.repo <<EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF

# Install the manager
sudo dnf install wazuh-manager -y
sudo systemctl enable --now wazuh-manager

Agent Enrollment

On the Wazuh Manager
sudo /var/ossec/bin/manage_agents

Press A to add an agent

Enter a name (e.g., SRV-2019)

Confirm and note the assigned Agent ID

Press E to extract the key — copy the base64 key

On the Windows Server (Endpoint)
Option A — Silent install with auto-enrollment:

Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi `
  -OutFile "$env:TEMP\wazuh-agent.msi"

msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /q `
  WAZUH_MANAGER="192.168.1.150" `
  WAZUH_AGENT_NAME="SRV-2019"

Start-Service -Name "WazuhSvc"

Option B — Manual key import:

cd "C:\Program Files (x86)\ossec-agent"
.\manage_agents.exe -i 001
Restart-Service -Name "WazuhSvc"

Custom Integration Script
Save as /var/ossec/integrations/custom-n8n.py:

#!/usr/bin/env python3
import sys
import json
import os
import requests

WEBHOOK_URL = "http://192.168.1.135:5678/webhook/wazuh-alerts"
LOG_FILE = "/var/ossec/logs/integrations.log"

def write_log(message):
    try:
        with open(LOG_FILE, "a") as f:
            f.write(f"[custom-n8n.py] {message}\n")
    except Exception:
        pass

def main():
    if len(sys.argv) < 2:
        write_log("[ERROR] Missing alert file argument.")
        sys.exit(1)

    alert_file_path = sys.argv[1]

    if not os.path.exists(alert_file_path):
        write_log(f"[ERROR] Alert file not found: {alert_file_path}")
        sys.exit(1)

    try:
        with open(alert_file_path, "r") as f:
            alert_data = json.load(f)
    except Exception as e:
        write_log(f"[ERROR] Failed to read alert JSON: {e}")
        sys.exit(1)

    try:
        response = requests.post(WEBHOOK_URL, json=alert_data, timeout=10)
        response.raise_for_status()
        write_log(f"[SUCCESS] Alert sent to n8n. Status: {response.status_code}")
    except requests.exceptions.RequestException as e:
        write_log(f"[ERROR] Failed to send alert to n8n: {e}")

if __name__ == "__main__":
    main()

    Set Permissions
sudo chmod 750 /var/ossec/integrations/custom-n8n.py
sudo chown root:wazuh /var/ossec/integrations/custom-n8n.py

Install Python Dependencies

sudo /var/ossec/framework/python/bin/pip3 install requests

Configure ossec.conf
Add this block inside the top-level <ossec_config> in /var/ossec/etc/ossec.conf, immediately before the final closing tag:

<integration>
  <name>custom-n8n.py</name>
  <hook_url>http://192.168.1.135:5678/webhook/wazuh-alerts</hook_url>
  <level>10</level>
  <alert_format>json</alert_format>
</integration>

⚠️ Important

Only one <ossec_config> root element is allowed. Do not duplicate.

The integration block must be a top-level child, not nested inside <syscheck> or <rootcheck>.

If editing via WinSCP + Notepad++, convert line endings:

sudo sed -i 's/\r$//' /var/ossec/etc/ossec.conf

Restart Wazuh
sudo systemctl restart wazuh-manager
sudo grep -i "integrator" /var/ossec/logs/ossec.log | tail -5

Expected:

wazuh-integratord: INFO: Enabling integration for: 'custom-n8n.py'.

Validation

Test the Script Manually

echo '{"rule": {"level": 12, "description": "Manual Script Test"}, "agent": {"name": "manual-test"}}' \
  | sudo tee /tmp/test_alert.json

sudo /var/ossec/framework/python/bin/python3 \
  /var/ossec/integrations/custom-n8n.py /tmp/test_alert.json

Expected:

[custom-n8n.py] [SUCCESS] Alert sent to n8n. Status: 200

Troubleshooting
Issue	Fix
integrations.log empty	Enable integrator daemon; check script permissions; verify <level> threshold
Error reading XML file (line 0)	Duplicate <ossec_config> tags or CRLF line endings
Connection refused in integrations.log	Check pfSense firewall rules for TCP/5678
Agent shows "Disconnected"	Restart WazuhSvc on Windows; verify <address> in ossec.conf

Useful Commands

# Service control
sudo systemctl restart wazuh-manager
sudo /var/ossec/bin/wazuh-control status

# List agents
sudo /var/ossec/bin/agent_control -l

# Live alerts
sudo tail -f /var/ossec/logs/alerts/alerts.json

# Integration log
sudo tail -f /var/ossec/logs/integrations.log


**Commit changes.**

✅ Done.

---

## 📄 File 5: `docs/n8n-setup.md`

**Add file → Create new file → `docs/n8n-setup.md`**

Paste:

```markdown
# ⚙️ n8n Setup & OAuth2 Configuration

Deploy n8n in Docker and connect it to the Gmail API via OAuth2 — including the private-IP workaround required for lab environments.

---

## Prerequisites

- Ubuntu Server 24.04 LTS VM (2 vCPU, 4 GB RAM, 30 GB disk)
- Static IP (e.g., `192.168.1.135`)
- Internet access
- A Gmail account

---

## Docker Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg

# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Verify
sudo docker run hello-world

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

---

## Generate an Encryption Key

Critical: n8n uses this key to encrypt credentials at rest. Back it up — losing it means losing access to all stored credentials.

openssl rand -base64 32
# Example: 1P0P5ESPXQJEM8SWN00S6CG2y/pATqLYMgZnELd128I=

## docker-compose.yml

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Asia/Manila
      - N8N_ENCRYPTION_KEY=PASTE_YOUR_KEY_HERE
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_SECURE_COOKIE=false
      - NODE_OPTIONS=--dns-result-order=ipv4first
      - N8N_HOST=localhost
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://192.168.1.135:5678/
    dns:
      - 1.1.1.1
      - 8.8.8.8
    extra_hosts:
      - "enterprise.n8n.io:104.26.13.187"
      - "enterprise.n8n.io:104.26.12.187"
      - "enterprise.n8n.io:172.67.68.102"
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:

## Environment Variables Explained

Variable	Purpose
GENERIC_TIMEZONE	Sets timezone for cron/schedule nodes
N8N_ENCRYPTION_KEY	Encrypts stored credentials at rest
N8N_SECURE_COOKIE=false	Allows login over HTTP (no HTTPS in lab)
N8N_HOST=localhost	Makes OAuth redirect URLs use localhost
WEBHOOK_URL	Base URL for webhooks — must be reachable by Wazuh
NODE_OPTIONS	Forces IPv4 DNS resolution

## Start n8n

docker compose up -d
docker compose logs -f n8n

Look for Editor is now accessible via: http://localhost:5678

## Google Cloud OAuth2 Setup

Why the SSH tunnel? Google blocks OAuth2 redirects to private IP addresses (RFC1918). We use an SSH tunnel so the browser sees localhost instead of 192.168.1.135.

Create a Google Cloud Project
Go to https://console.cloud.google.com

Create a new project (e.g., n8n-SOAR-Lab)

Select the project

2. Enable the Gmail API
APIs & Services → Library

Search Gmail API → Enable

3. Configure OAuth Consent Screen
APIs & Services → OAuth consent screen

Choose External → Create

App name: n8n Lab, fill in your email fields

Scopes: click Save and Continue

Test users: click + ADD USERS and add your Gmail address ⚠️ Critical — the app will be blocked otherwise

4. Create OAuth2 Credentials
APIs & Services → Credentials → + CREATE CREDENTIALS → OAuth client ID

Application type: Web application

Name: n8n Gmail

Authorized redirect URI:

http://localhost:5678/rest/oauth2-credential/callback

Click Create → copy Client ID and Client Secret

5. Start the SSH Tunnel

From your Windows machine:

ssh -L 5678:localhost:5678 socpanda@192.168.1.135

Keep this window open. Access n8n at http://localhost:5678.

6. Connect Gmail in n8n

Add a Gmail node → Create new credential

Verify the OAuth Redirect URL shows http://localhost:5678/rest/oauth2-credential/callback

Paste your Client ID and Client Secret

Click Sign in with Google → choose your account

If prompted about an unverified app: Advanced → Go to n8n (unsafe) → Allow

You should see "Connection tested successfully" → Save

Building the Workflow

[ Webhook ] → [ IF ] → [ Edit Fields ] → [ Gmail ]

Node 1 — Webhook
Field	Value
HTTP Method	POST
Path	wazuh-alerts
Respond	Immediately
Node 2 — IF
Field	Value
Value 1	{{ $json.body.rule.level }} (Expression)
Operation	Number → Larger Equal
Value 2	10
Convert types	ON
Node 3 — Edit Fields (Set)
Field Name	Value (Expression)
EmailSubject	Wazuh Alert: {{ $json.body.rule.description }}
Severity	{{ $json.body.rule.level }}
AgentName	{{ $json.body.agent.name }}
Description	{{ $json.body.rule.description }}
Node 4 — Gmail
Field	Value
Credential	Your Gmail OAuth2 credential
Resource	Message
Operation	Send
To	your.email@gmail.com
Subject	{{ $json.EmailSubject }}
Email Type	HTML
Message body:

<p><strong>Severity:</strong> {{ $json.Severity }}</p>
<p><strong>Agent:</strong> {{ $json.AgentName }}</p>
<p><strong>Description:</strong> {{ $json.Description }}</p>

Activate
Toggle the switch in the top-right corner to Active.

Troubleshooting
Error	Fix
redirect_uri_mismatch	Copy OAuth Redirect URL from n8n exactly into Google Cloud; no trailing slash
device_id and device_name required for private IP	Use the SSH tunnel and access n8n via localhost
Mismatching encryption keys	Never change N8N_ENCRYPTION_KEY. For a fresh lab, wipe the volume
localhost refused to connect	SSH tunnel is not running
UI resets to setup page	Cookie host mismatch — clear cookies for both localhost and the IP


Useful Commands
docker compose restart
docker compose logs -f n8n
docker exec -it n8n sh
docker run --rm -v n8n-compose_n8n_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/n8n_backup.tar.gz /data

  
**Commit changes.**

✅ Done.

---

## 📄 File 6: `integrations/custom-n8n.py`

**Add file → Create new file → `integrations/custom-n8n.py`**

Paste:

```python
#!/usr/bin/env python3
"""
Wazuh -> n8n integration script.

Invoked by the Wazuh integratord daemon when an alert matches the
configured severity threshold. Reads the alert JSON file passed as
argv[1] and forwards it to an n8n webhook for automated triage and
email notification.

Author  : SOC_Panda
Project : Wazuh -> n8n -> Gmail SOAR Pipeline
License : MIT
"""

import sys
import json
import os
import requests

# -----------------------------------------------------------------------------
# Configuration
# -----------------------------------------------------------------------------
WEBHOOK_URL = "http://192.168.1.135:5678/webhook/wazuh-alerts"
LOG_FILE = "/var/ossec/logs/integrations.log"


# -----------------------------------------------------------------------------
# Helpers
# -----------------------------------------------------------------------------
def write_log(message: str) -> None:
    """Append a timestamped message to the integration log."""
    try:
        with open(LOG_FILE, "a") as f:
            f.write(f"[custom-n8n.py] {message}\n")
    except Exception:
        pass


# -----------------------------------------------------------------------------
# Main
# -----------------------------------------------------------------------------
def main() -> None:
    if len(sys.argv) < 2:
        write_log("[ERROR] Missing alert file argument.")
        sys.exit(1)

    alert_file_path = sys.argv[1]

    if not os.path.exists(alert_file_path):
        write_log(f"[ERROR] Alert file not found: {alert_file_path}")
        sys.exit(1)

    try:
        with open(alert_file_path, "r") as f:
            alert_data = json.load(f)
    except Exception as e:
        write_log(f"[ERROR] Failed to read or parse alert JSON: {e}")
        sys.exit(1)

    try:
        response = requests.post(WEBHOOK_URL, json=alert_data, timeout=10)
        response.raise_for_status()
        write_log(f"[SUCCESS] Alert sent to n8n. Status: {response.status_code}")
    except requests.exceptions.RequestException as e:
        write_log(f"[ERROR] Failed to send alert to n8n: {e}")


if __name__ == "__main__":
    main()

```
Commit changes.

✅ Done.
---
📄 File 7: configs/docker-compose.yml

Add file → Create new file → configs/docker-compose.yml

Paste:

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Asia/Manila
      - N8N_ENCRYPTION_KEY=CHANGE_ME
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_SECURE_COOKIE=false
      - NODE_OPTIONS=--dns-result-order=ipv4first
      - N8N_HOST=localhost
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://192.168.1.135:5678/
    dns:
      - 1.1.1.1
      - 8.8.8.8
    extra_hosts:
      - "enterprise.n8n.io:104.26.13.187"
      - "enterprise.n8n.io:104.26.12.187"
      - "enterprise.n8n.io:172.67.68.102"
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:

  Commit changes.

✅ Done.

📄 File 8: configs/ossec-integration.xml
Add file → Create new file → configs/ossec-integration.xml

Paste:

<!--
  Wazuh Integration Snippet
  ============================================================
  Add this block inside the top-level <ossec_config> element in
  /var/ossec/etc/ossec.conf, immediately before the final closing tag.

  Fields:
    <name>          Must match the script filename exactly
    <hook_url>      The n8n production webhook URL
    <level>         Minimum severity to trigger the integration (1-16)
    <alert_format>  Must be "json" so the script can parse the alert
  ============================================================
-->
<integration>
  <name>custom-n8n.py</name>
  <hook_url>http://192.168.1.135:5678/webhook/wazuh-alerts</hook_url>
  <level>10</level>
  <alert_format>json</alert_format>
</integration>

```
Commit changes.

✅ Done.










