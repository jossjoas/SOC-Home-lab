Build a Mini-SOC with Wazuh — Full Deep-Dive Lab Guide
________________________________________
📦 COMPONENTS & TOPOLOGY
You'll be integrating:
•	✅ Ubuntu VM (Wazuh Manager, Elasticsearch, Kibana)
•	✅ Windows Server 2022 (AD, Sysmon, Wazuh Agent)
•	✅ Windows Workstation (Wazuh Agent + Sysmon)
•	✅ pfSense (Firewall logs to Wazuh via Syslog)
•	✅ CrowdSec (Optional integration)
•	✅ VirtualBox as the hypervisor
•	✅ Internal Network: GREEN (used for SOC communications)
________________________________________
🖥️ PART 1: Creating the Ubuntu Wazuh Manager VM
________________________________________
🧱 Step 1.1: Create Ubuntu VM in VirtualBox
1.	Download Ubuntu Server LTS
o	URL: https://ubuntu.com/download/server
o	Use Ubuntu Server 22.04 LTS (64-bit) ISO
2.	Create a new VM in VirtualBox
o	Name: Wazuh-Manager
o	Type: Linux
o	Version: Ubuntu (64-bit)
3.	Assign resources
o	Memory: 4096 MB minimum (better: 6144 MB or 8192 MB)
o	CPUs: 2 minimum (4 recommended)
o	Storage: Create virtual disk
	Type: VDI
	Size: 40 GB dynamically allocated
4.	Attach Network
o	Adapter 1: Internal Network GREEN (connects to Server & Workstation)
o	Adapter 2: (Optional) NAT or Bridged (for internet access or external updates)
5.	Mount the ISO
o	In Settings > Storage > Controller IDE, attach the Ubuntu ISO
________________________________________
🧾 Step 1.2: Install Ubuntu
1.	Boot the VM
2.	Select “Install Ubuntu Server”
3.	Follow prompts:
o	Language: English
o	Keyboard layout: US
o	Network: auto-detect Internal IP (or set static if desired)
o	Hostname: wazuh
o	Username: wazuhadmin, password: your choice
o	Storage: use entire disk (auto partition)
o	Enable OpenSSH server
o	Skip Snap packages (unless needed)
4.	Reboot after install
5.	Log in and run initial updates:
bash
CopyEdit
sudo apt update && sudo apt upgrade -y
________________________________________
🧠 PART 2: Installing Wazuh All-In-One Stack
________________________________________
🔧 Step 2.1: Install Wazuh with the official script
This installs:
•	Wazuh Manager
•	Filebeat (for log shipping)
•	Elasticsearch
•	Kibana
•	Wazuh Dashboards
bash
CopyEdit
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
bash wazuh-install.sh -a
•	During the script, you’ll be asked to set the admin password for the Wazuh dashboard.
•	The install takes ~10–20 minutes (depending on resources).
________________________________________
🔍 Step 2.2: Verify installation
•	Access dashboard from browser:
url
CopyEdit
https://<wazuh-VM-IP>/
•	Credentials:
o	Username: admin
o	Password: what you chose in the script
•	Test login and open the dashboard.
________________________________________
👮 PART 3: Install Wazuh Agents on Windows Systems
________________________________________
🖥️ Step 3.1: Download Windows Agent
From:
https://packages.wazuh.com/
Download the latest .msi agent installer on both Windows Server and Workstation.
________________________________________
🔧 Step 3.2: Install Agent with Wazuh Manager IP
GUI Method
•	Run the .msi on each Windows host
•	Set:
o	Wazuh Manager IP: your Ubuntu VM internal IP (e.g., 192.168.100.100)
o	Agent name: e.g., WIN-SERVER, WIN-WS01
CLI Silent Install (PowerShell or CMD)
c
CopyEdit
msiexec /i "wazuh-agent-4.7.x.msi" /qn WAZUH_MANAGER="192.168.100.100" WAZUH_AGENT_NAME="WIN-SERVER"
________________________________________
▶️ Step 3.3: Start & Register Agent
powershell
CopyEdit
net start wazuh
•	Check that the agent connects on the Wazuh dashboard under “Agents”
•	Once registered, the agent will start sending logs
________________________________________
📜 PART 4: Enable Log Sources & Monitoring
________________________________________
🪓 Step 4.1: Enable Sysmon on Windows Hosts
1.	Install Sysmon from Sysinternals
o	https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
2.	Use community config for detection coverage (e.g. SwiftOnSecurity):
o	Download config:
bash
CopyEdit
https://github.com/SwiftOnSecurity/sysmon-config
3.	Install:
powershell
CopyEdit
.\sysmon.exe -accepteula -i sysmonconfig.xml
This enables process creation, file creation, network events, etc.
________________________________________
🔍 Step 4.2: Enable Windows Event Channels
On the Wazuh Manager:
bash
CopyEdit
sudo nano /var/ossec/etc/ossec.conf
Add inside <localfile>:
xml
CopyEdit
<localfile>
  <location>eventchannel</location>
  <log_format>eventchannel</log_format>
  <eventchannel>Security</eventchannel>
  <eventchannel>System</eventchannel>
  <eventchannel>Application</eventchannel>
  <eventchannel>Microsoft-Windows-Sysmon/Operational</eventchannel>
</localfile>
Restart Wazuh:
bash
CopyEdit
sudo systemctl restart wazuh-manager
________________________________________
🌐 PART 5: Integrate pfSense Logs into Wazuh
________________________________________
🔀 Step 5.1: Configure pfSense Remote Syslog
1.	Go to pfSense Web UI → Status > System Logs > Settings
2.	Enable Remote Logging
o	IP: Wazuh Manager IP (GREEN interface)
o	Port: 514 (UDP)
o	Facility: local0
o	Select logs: System, Firewall, DHCP, etc.
________________________________________
🛠️ Step 5.2: Configure rsyslog on Ubuntu
Install and configure rsyslog:
bash
CopyEdit
sudo apt install rsyslog -y
sudo nano /etc/rsyslog.d/10-pfsense.conf
Add:
c
CopyEdit
module(load="imudp")
input(type="imudp" port="514")

if ($fromhost-ip == '192.168.100.1') then {
  action(type="omfile" file="/var/log/pfsense.log")
}
Restart:
bash
CopyEdit
sudo systemctl restart rsyslog
________________________________________
⚙️ Step 5.3: Configure Wazuh to Monitor pfSense Logs
Edit /var/ossec/etc/ossec.conf and add:
xml
CopyEdit
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/pfsense.log</location>
</localfile>
Restart:
bash
CopyEdit
sudo systemctl restart wazuh-manager
________________________________________
🛡️ PART 6: Verify Logs & Alerts
________________________________________
In Wazuh Dashboard:
•	Go to “Agents” → Select an agent
o	Check Security Events tab
o	Confirm Sysmon logs, logons, failed attempts, etc.
•	Go to “Security Events” for:
o	RDP brute force
o	Powershell exec
o	Suspicious child processes
o	Sysmon Event ID 1, 3, 10, etc.
________________________________________
🧪 PART 7: Test Detection and Response
________________________________________
Try these simulations:
Action	Detection
PowerShell execution	Sysmon Event ID 1
Brute-force logins	Windows Event ID 4625
File modification	Sysmon FileCreate
Ransomware behavior	Wazuh rules (modify ransom notes)
________________________________________
🧰 PART 8: Optional CrowdSec Integration
1.	Install CrowdSec on your Windows Hosts
2.	Forward logs to Wazuh using filebeat or syslog
3.	Alternatively, monitor CrowdSec logs via Wazuh
________________________________________
🧾 PART 9: Summary
Component	Purpose
Ubuntu VM	Wazuh stack host
Windows Server	AD + Wazuh Agent
Windows Workstation	Client + Wazuh Agent
pfSense	Syslog source
Sysmon	Low-level system logging
CrowdSec	Real-time

