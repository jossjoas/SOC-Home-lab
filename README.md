Build a Mini-SOC with Wazuh — Full Deep-Dive Lab Guide
1. Components & Topology

Integrated components:

Ubuntu VM — Wazuh Manager, Elasticsearch, Kibana

Windows Server 2022 — Active Directory, Sysmon, Wazuh Agent

Windows Workstation — Wazuh Agent + Sysmon

pfSense — Firewall logs forwarded to Wazuh via Syslog

CrowdSec (optional integration)

VirtualBox as the hypervisor

Internal Network: GREEN (SOC communications)

2. Creating the Ubuntu Wazuh Manager VM
Step 2.1: Create Ubuntu VM in VirtualBox

Download Ubuntu Server 22.04 LTS (64-bit):
https://ubuntu.com/download/server

Create a new VM:

Name: Wazuh-Manager

Type: Linux

Version: Ubuntu (64-bit)

Allocate resources:

Memory: 4 GB minimum (6–8 GB recommended)

CPU: 2 minimum (4 recommended)

Disk: 40 GB (VDI, dynamically allocated)

Networking:

Adapter 1: Internal Network GREEN

Adapter 2: (Optional) NAT or Bridged

Mount ISO in VirtualBox under Settings > Storage.

Step 2.2: Install Ubuntu

Boot VM and install Ubuntu Server.

Configure:

Language: English

Network: Internal IP or static if preferred

Hostname: wazuh

User: wazuhadmin (set your password)

Enable OpenSSH server

Reboot and update:

sudo apt update && sudo apt upgrade -y

3. Installing Wazuh All-In-One Stack
Step 3.1: Install Wazuh with the Official Script

This will install:

Wazuh Manager

Filebeat

Elasticsearch

Kibana

Wazuh Dashboards

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
bash wazuh-install.sh -a


You will be prompted to set the admin password during installation.

Step 3.2: Verify Installation

Access: https://<wazuh-VM-IP>/

Login:

Username: admin

Password: (set during installation)

4. Installing Wazuh Agents on Windows Systems
Step 4.1: Download Windows Agent

Download the latest MSI from:
https://packages.wazuh.com/

Step 4.2: Install Agent

GUI:

Run the MSI

Wazuh Manager IP: 192.168.100.100 (example)

Agent name: WIN-SERVER or WIN-WS01

CLI (Silent Install):

msiexec /i "wazuh-agent-4.7.x.msi" /qn WAZUH_MANAGER="192.168.100.100" WAZUH_AGENT_NAME="WIN-SERVER"

Step 4.3: Start & Register Agent
net start wazuh


Confirm connection in the Wazuh dashboard under Agents.

5. Enabling Log Sources & Monitoring
Step 5.1: Install Sysmon on Windows

Download Sysmon:
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

Use a recommended config (e.g., SwiftOnSecurity):
https://github.com/SwiftOnSecurity/sysmon-config

Install:

.\sysmon.exe -accepteula -i sysmonconfig.xml

Step 5.2: Enable Windows Event Channels on Wazuh Manager

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

6. Integrating pfSense Logs
Step 6.1: Configure pfSense Remote Syslog

Go to Status > System Logs > Settings

Enable Remote Logging:

IP: Wazuh Manager (GREEN)

Port: 514 UDP

Facility: local0

Logs: System, Firewall, DHCP, etc.

Step 6.2: Configure rsyslog on Ubuntu
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

Step 6.3: Monitor pfSense Logs in Wazuh
sudo nano /var/ossec/etc/ossec.conf


Add:

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/pfsense.log</location>
</localfile>


Restart:

sudo systemctl restart wazuh-manager

7. Verifying Logs and Alerts

In the Wazuh Dashboard:

Agents → Select agent → Security Events

Check Sysmon logs, logons, failed attempts, etc.

Security Events → Review detections:

RDP brute force

PowerShell execution

Suspicious processes

Sysmon Events (IDs 1, 3, 10, etc.)
