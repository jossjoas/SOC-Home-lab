🛡️ Build a Mini-SOC with Wazuh — Full Deep-Dive Lab Guide
🧭 1. Components & Topology

You’ll integrate:

🐧 Ubuntu VM — Wazuh Manager, Elasticsearch, Kibana

🖥️ Windows Server 2022 — AD, Sysmon, Wazuh Agent

💻 Windows Workstation — Wazuh Agent + Sysmon

🧱 pfSense — Firewall logs via Syslog

🧠 CrowdSec (optional) — Behavioral detection

🧰 VirtualBox — Hypervisor

🟢 Internal Network (GREEN) — SOC traffic

🖥️ 2. Creating the Ubuntu Wazuh Manager VM
🧱 Step 2.1: Create Ubuntu VM

📥 Download Ubuntu Server 22.04 LTS:
👉 https://ubuntu.com/download/server

⚙️ VM Configuration:

Name: Wazuh-Manager

Type: Linux / Ubuntu (64-bit)

RAM: 4–8 GB

CPU: 2–4 cores

Disk: 40 GB (VDI)

🌐 Networking:

Adapter 1: Internal Network (GREEN)

Adapter 2: (Optional) NAT / Bridged

💿 Mount ISO in VirtualBox (Settings → Storage).

🧾 Step 2.2: Install Ubuntu

Language: English

Hostname: wazuh

User: wazuhadmin (set password)

Enable OpenSSH

Reboot and update:

sudo apt update && sudo apt upgrade -y

⚡ 3. Installing Wazuh All-In-One Stack
🧰 Step 3.1: Installation Script

Installs Wazuh Manager, Filebeat, Elasticsearch, Kibana, Dashboards:

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
bash wazuh-install.sh -a


📝 Set the admin password when prompted.

🔍 Step 3.2: Access Dashboard

URL: https://<WAZUH-VM-IP>/

👤 User: admin

🔑 Password: chosen during install

🧑‍💻 4. Installing Wazuh Agents on Windows
📥 Step 4.1: Download

👉 https://packages.wazuh.com/

🧭 Step 4.2: Install

GUI:

Manager IP: 192.168.100.100

Agent name: WIN-SERVER or WIN-WS01

CLI Silent Install:

msiexec /i "wazuh-agent-4.7.x.msi" /qn WAZUH_MANAGER="192.168.100.100" WAZUH_AGENT_NAME="WIN-SERVER"

🚀 Step 4.3: Start & Register Agent
net start wazuh


✅ Check in Wazuh Dashboard → Agents.

🪓 5. Enabling Sysmon & Event Channels
🧰 Step 5.1: Install Sysmon

Download: Sysinternals Sysmon

Use community config: SwiftOnSecurity

.\sysmon.exe -accepteula -i sysmonconfig.xml

🧭 Step 5.2: Configure Event Channels

Edit:

sudo nano /var/ossec/etc/ossec.conf


Add:

<localfile>
  <location>eventchannel</location>
  <log_format>eventchannel</log_format>
  <eventchannel>Security</eventchannel>
  <eventchannel>System</eventchannel>
  <eventchannel>Application</eventchannel>
  <eventchannel>Microsoft-Windows-Sysmon/Operational</eventchannel>
</localfile>


Restart:

sudo systemctl restart wazuh-manager

🌐 6. pfSense Integration
🛜 Step 6.1: Remote Syslog in pfSense

✅ Enable Remote Logging

IP: Wazuh Manager (GREEN)

Port: 514 UDP

Facility: local0

Logs: System, Firewall, DHCP…

🐧 Step 6.2: Configure rsyslog on Ubuntu
sudo apt install rsyslog -y
sudo nano /etc/rsyslog.d/10-pfsense.conf


Add:

module(load="imudp")
input(type="imudp" port="514")

if ($fromhost-ip == '192.168.100.1') then {
  action(type="omfile" file="/var/log/pfsense.log")
}


Restart:

sudo systemctl restart rsyslog

🛡️ Step 6.3: Add to Wazuh
sudo nano /var/ossec/etc/ossec.conf

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/pfsense.log</location>
</localfile>

sudo systemctl restart wazuh-manager

🧪 7. Verify Logs & Alerts

📊 In Wazuh Dashboard:

Agents → Select agent → Security Events

Sysmon logs

Logons

Failed attempts

Security Events:

RDP brute force

PowerShell exec

Suspicious child processes

Sysmon Events (ID 1, 3, 10…)

🔥 8. Detection & Response Testing
🧪 Action	🕵️ Detection
PowerShell execution	Sysmon Event ID 1
Brute-force logins	Windows Event ID 4625
File modification	Sysmon FileCreate
Ransomware activity	Wazuh rules & alerts
